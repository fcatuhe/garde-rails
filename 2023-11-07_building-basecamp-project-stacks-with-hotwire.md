# Building Basecamp project stacks<br /> with Hotwire

**Author:** Nicklas Ramhöj Holtryd, Programmer, Product
**Published:** November 7, 2023
**Source:** <https://dev.37signals.com/building-basecamp-project-stacks-with-hotwire/>

---

It’s been two decades since Rails changed the way we build web apps. As the demand for richer and richer UIs grew, teams came up with different frontends to deliver on those expectations. Client-side frameworks such as Angular, Ember, and React emerged as popular choices.

One of Rails’ strengths has always been to empower individual programmers to not just contribute to, but to ship entire full stack applications themselves. This isn’t just great for small teams but can have a radical impact on larger organizations’ communication overhead.

While spending months on coming up with the most performant way to render a login screen might have been viable during the era of [zero interest rates](https://youtu.be/iqXjGiQ_D-A?si=BR-gCpdPuZ6FWcns&t=719), I’d like to think those times are over.

At 37signals we work in teams of two, one designer, one programmer; and we ship projects in 6 weeks or less. Working full stack and utilizing progressive enhancement is crucial to maintaining our productivity. There’s a lot [more to it](https://dev.37signals.com/the-10x-development-environment/) but without [Hotwire](https://hotwired.dev/) at our disposal things would look a lot different.

In this blog post I want to show you how we used [Turbo](https://turbo.hotwired.dev/) and [Stimulus](https://stimulus.hotwired.dev/) to implement project [stacks](https://updates.37signals.com/post/new-in-basecamp-organize-projects-into-stacks), a new feature to visually group projects in [Basecamp](https://basecamp.com/).

---

[📺 Watch video](https://videos.37signals.com/dev/assets/videos/building-basecamp-project-stacks-with-hotwire/walkthrough.mp4)

This video shows what we’ll build towards in this blog post and provide context to the code that follows. You can skip over it if you’re already familiar with Stacks in Basecamp. Optionally you’re also welcome to check out the behind the scenes [design review](https://www.youtube.com/watch?v=ecctvU_9I0c) for more details on our UX considerations.

Stacks UI walkthrough

The video shows how Projects can be moved into Stacks on the Basecamp home screen. The following actions are demonstrated in it and following videos:
- Create a new Stack by dragging a pinned Project over another pinned Project.
- Name Stacks in a modal during creation.
- Close a Stack modal.
- Stack a Project by dropping a Project over an existing Stack.
- Create a new Stack by dragging a recent Project over a Pinned project.
- Stack’s project counter.
- Sort Projects and Stacks among themselves using drag-and-drop.
- Empty and delete a Stack by first clicking it’s name.
- Unstack a Project by dragging it out of the open Stack modal.

---

## Sorting using delegated types

In Basecamp users can Pin Projects, making them show up on the landing page. A Pin has a position which allows Projects to be sorted. While we don’t yet have a way to add Stacks (think of them as Project folders) I wanted to make sure we had a plan for how to sort Project and Stack among each other.

Project is modeled as a Delegated Type, to use the [documentation example](https://api.rubyonrails.org/classes/ActiveRecord/DelegatedType.html) Projects in Basecamp are Entryables (we call them Bucketables) and our Entry is a Bucket. By modeling Stack as a Bucketable we get the pinning behavior for free.

![Modeling diagram](https://dev.37signals.com/assets/images/building-basecamp-project-stacks-with-hotwire/modeling.png)

Modeling details

A diagram showing the data models, mapping to database tables tables with the schema changes highlighted:
- Existing tables:
- `Bucket::Pin`: `belongs_to` `Bucket` and `User`, also has a position field.
- `Bucket`: The Delegated Types wrapper class which delegates to `Project` and `Stack`.
- `Project`: Our existing `Bucketable` next to the new one: `Stack`.
- `Person` / `User`: Another Delegated Type, we won’t get into this distinction here.
- New model: `Stack` which `belongs_to` `Person` as `creator`.

-

New field: `parent_id` on `bucket_pins`, for a self-referencing parent-child relation.
- Design consequences:
- `Stack` as a new `Bucketable`.
- `Bucket::Pin` as a `Tree` node.
- `User` specific due to `Project` belonging to `Stack` via `Bucket::Pin`.
- Minimal schema changes and no renames.
- Can sort Stacks and Projects with existing algorithm.

Abridged ActiveRecord definitions:

```
class Bucket < ApplicationRecord
  include Pinnable
  delegated_type :bucketable, types: %w[ Project Stack ]
end

module Bucket::Pinnable
  has_many :pins, dependent: :destroy
  scope :ordered, -> { joins(:pins).merge(Bucket::Pin.ordered) }
end

class Project < ApplicationRecord
  include Bucketable
end

class Stack < ApplicationRecord
  include Bucketable
end

module Bucketable
  has_one :bucket, as: :bucketable
end

class User < ApplicationRecord
  include BucketPinner
end

module User::BucketPinner
  has_many :bucket_pins, class_name: "Bucket::Pin"
  has_many :pinned_buckets, -> { Bucket::Pin.ordered }, class_name: "Bucket", source: :bucket, through: :bucket_pins
end

class Bucket::Pin < ApplicationRecord
  include Positioned

  belongs_to :user
  belongs_to :bucket
end

module Positioned
  scope :ordered, -> { order(:position) }
end
```

Now when we render pinned buckets they include Buckets with both Project and Stack Bucketables. Since Bucket takes care of positioning via the `Positioned` concern, sorting works independently of the Bucketable.

```
<% Current.user.pinned_buckets.each do |bucket| %>
  <%= render bucket.to_bucketable_partial_path, bucket: bucket %>
<% end %>
```

---

## User-independent bucket nesting

So far so good, but the whole point of a Stack is to hold Projects so we need a way to tell in what Stack a given project resides. We can’t add `parent_id` to Bucket because then one user’s organization would affect everyone. Instead let’s add it to the Pin model. Pins are already User specific and are responsible for positioning so it makes sense for it to also hold the container reference. Or in other words, know in what Stack it’s pinned.

We already have a basic Tree concern so let’s include it:

```
class Bucket::Pin < ApplicationRecord
  include Positioned, Tree
end

module Tree
  extend ActiveSupport::Concern

  included do |base|
    belongs_to :parent, class_name: base.to_s, touch: true, optional: true, inverse_of: :children
    has_many :children, class_name: base.to_s, foreign_key: :parent_id, dependent: :destroy, inverse_of: :parent

    scope :roots, -> { where(parent: nil) }
    scope :children_of, ->(parent) { where(parent: parent) }
  end

  def adopt(children)
    transaction do
      Array(children).each do |child|
        child.update! parent: self
      end
    end
  end
end
```

We can now nest pins. To exclude Stacked Projects from being rendered as “roots” all we have to do is to modify the scope a bit:

```
<% Current.user.pinned_buckets.merge(Bucket::Pin.roots).each do |bucket| %>
  …
<% end >
```

And when we click a Stack we’ll render only that Stack’s Projects:

```
class Stack < ApplicationRecord
  include Bucketable
  belongs_to :creator, class_name: "Person", default: -> { Current.person }

  def buckets
    pin = bucket.pins.find_sole_by(user_id: creator.user_id)
    Bucket.ordered.merge Bucket::Pin.children_of(pin)
  end
end
```

```
<!-- stacks/show.html.erb -->
<%= render partial: "stacks/project", collection: @stack.buckets, as: :bucket %>
```

Next up we need to modify `Positioned` to be aware of `parent_id`. If not, stacked Projects would mess up the positions of our unstacked Projects and Stacks.

```
module Positioned
  …
  class_methods do
    def shift_positions_upward
      shift_positions_by(-1)
    end

    def shift_positions_downward
      shift_positions_by(+1)
    end

    def shift_positions_by(change)
      update_all([ "position = position + ?", change ])
    end
  end

  def reposition_to(position_identifier, new_parent: nil)
    new_position = numeric_position_from(position_identifier, new_parent:)

    with_lock do
      case
      when new_parent && new_parent != parent # Stack project
        reposition_in_tree new_parent: new_parent, new_position: 1
      when new_parent == false && parent # Unstack project
        reposition_in_tree new_parent: nil, new_position: new_position
      when new_position != position # Reposition at current nesting level
        update_position new_position
      end
    end
  end

  private
    def reposition_in_tree(new_parent:, new_position:)
      subsequent_positionings.shift_positions_upward
      update! parent: new_parent, position: new_position
      subsequent_positionings.shift_positions_downward
    end

    def subsequent_positionings
      positioned_scope.excluding(self).where position: position..
    end

    def numeric_position_from(position_identifier)
      # Converts things like "bottom" to a numeric position and checks for out of bounds input.
    end
    …
end

class Bucket::Pin < ApplicationRecord
  …
  def positioned_scope
    user_bucket_pins.where parent_id:
  end
end
```

---

## Nested sorting via drag-and-drop

That’s the basic modeling in place. There are other concerns such as updating positions when things are destroyed or created but we won’t get into those here. Instead let’s turn our attention to the UI.

![Docs & Files with Folders](https://dev.37signals.com/assets/images/building-basecamp-project-stacks-with-hotwire/docs-and-files.png)

Docs & Files with Folders

![Home screen with Projects and Stacks](https://dev.37signals.com/assets/images/building-basecamp-project-stacks-with-hotwire/stacks.png)

Home screen with Projects and Stacks

Projects in Basecamp already have a drag-and-drop sort UI and because we use the same system to manage folders in the Docs & Files tool we can even use it for nesting. It works great but the Javascript was written a long time ago using jQuery and conventions that we wouldn’t use today.

[📺 Watch video](https://videos.37signals.com/dev/assets/videos/building-basecamp-project-stacks-with-hotwire/stack-project.mp4)

Drag-and-drop is one of those things that is easy to get up and running but difficult to get just right. Minding the project’s [appetite](https://basecamp.com/shapeup/1.2-chapter-03#setting-the-appetite) I decided to use what we got, but only if I could do so in a way that didn’t add any new complexity or couplings to the legacy system.

Upon a drop event, `position` and `parent_id` (if any) are sent to the server. Since we’ve already taken care of nested positioning in the `Positioned` concern we just have to pass on the new parent to `reposition_to`.

```
class Buckets::Pins::PositionsController < ApplicationController
  before_action :set_bucket_pin, :set_parent

  def update
    @bucket_pin.reposition_to params.require(:position), new_parent: @parent
  end

  private
    def set_bucket_pin
      @bucket_pin = Current.user.bucket_pins.find_sole_by(bucket_id: params.require(:bucket_id))
    end

    def set_parent
      if parent_id_param.root?
        @parent = false
      else parent_id_param.present?
        @parent = Current.person.buckets.find(parent_id_param)
      end
    end

    def parent_id_param
      params.fetch(:parent_id, "").inquiry
    end
end
```

---

## Creating Stacks

So far we’ve dropped Projects into Stacks that we’ve created via the Rails console. Let’s make it real by adding a UI where a Project can be dropped on another Project to form a new Stack. This use case wasn’t considered by the existing Javascript but by tagging project Elements as folders it will send the project’s `bucket_id` as `parent_id`, no Javascript changes required.

[📺 Watch video](https://videos.37signals.com/dev/assets/videos/building-basecamp-project-stacks-with-hotwire/create-stack.mp4)

Let’s update our Rails controller to handle Stack creation:

```
class Buckets::Pins::PositionsController < ApplicationController
  …
  def create
    if should_form_new_stack?
      @stack = Stack.amalgamate(@bucket_pin, into: @parent)
    else
      @bucket_pin.reposition_to position_param, new_parent: @parent
      @stack = @bucket_pin.current_or_previous_bucket&.stack
    end
  end

  private
    def should_form_new_stack?
      @parent.present? && @parent.bucket.project?
    end
    helper_method :should_form_new_stack?
    …
end

class Stack < ApplicationRecord
  class << self
    def amalgamate(*children, into:)
      create!.tap do |stack|
        insert_at = into.position # Remember position pre-adoption.
        stack.pin.adopt [ into ] + children
        stack.pin.reposition_to insert_at
      end
    end
  end
end
```

Great! A new Stack is created and all the positions are correctly updated to reflect the new state, after a manual page reload everything looks correct.

---

## Updating the UI

So how do we actually update the UI? Obviously we can’t expect users to reload the page after dropping an Element.

### Enhance with Turbo Streams

If you’ve used Turbo before you know how well it lends itself to progressive enhancement. Out of the box you get faster page reloads with Turbo Drive and you can optimize further with Turbo Frames, Turbo Streams, and Stimulus controllers. Let’s have a look at how we can use Turbo to enhance existing Javascript as well.

Because our Javascript uses `Turbo.renderStreamMessage` it will act on what we return from `buckets/pins/positions/create.turbo_stream.erb`. We’ll use this to define the Stack specific behavior we’re looking for, without changing the sortable Javascript.

```
<%= turbo_stream.replace dom_id(@parent.bucket), partial: "stacks/stack",
      locals: { bucket: @stack.bucket } if should_form_new_stack? %>
```

Now when a Project is dropped on another we’ll replace the Project dropped on with the newly created Stack. This re-uses all the server side logic we need for the initial render and isn’t coupled to any of the Javascript that issued the Stack creation request.

To update the Stack’s projects counter we’ll add another one-liner that replaces the Stack Element:

```
<%= turbo_stream.replace dom_id(@stack), partial: "stacks/stack",
      locals: { bucket: @stack.bucket } if @stack %>
```

### Unstacking with Stimulus

So far we can sort and we can move a Project to a Stack or to another Project to form a new Stack. Let’s implement unstacking a Project.

[📺 Watch video](https://videos.37signals.com/dev/assets/videos/building-basecamp-project-stacks-with-hotwire/unstack-project.mp4)

We opted for unstacking when a Project is dropped anywhere outside the Stack modal. This is a new drag-and-drop interaction not covered by the legacy Javascript. Luckily it’s a simple one:

```
import { Controller } from "@hotwired/stimulus"
import { request } from "../helpers/request_helpers"

export default class extends Controller {
  static targets = [ "project" ]

  setupDraggedElement(event) {
    event.dataTransfer.setData("text/plain", event.target.id)
  }

  acceptDrop(event) {
    if (this.element === event.target) {
      const element = this.#getDraggedElement(event)
      this.#unstackProject(element)
    }
  }

  // Private

  #getDraggedElement(event) {
    const draggedElementId = event.dataTransfer.getData("text/plain")
    return this.projectTargets.find(target => target.id === draggedElementId)
  }

  #unstackProject(element) {
    this.#postUpdate(element.dataset.url)
    element.remove()
  }

  #postUpdate(url) {
    const body = new FormData()
    body.append("parent_id", "root")
    body.append("position", "bottom")

    request.post(url, { body })
  }
}
```

```
<!-- stacks/show.html.erb -->
<bc-modal id="stack_modal" data-controller="stack" data-action="drop->stack#acceptDrop">
  …
  <%= render partial: "stacks/project", collection: @stack.buckets %>
</bc-modal>

<!-- stacks/_project.html.erb -->
<article data-stack-target="project" data-action="dragstart->stack#setupDraggedElement">
  …
</article>
```

Now when we drop a project Element on the modal background we’ll send a request to our pin positions endpoint which will unstack the Project. To make it appear among it’s new siblings we’ll simply add another one-liner to our Turbo Stream response:

```
<%= turbo_stream.append :pinned_buckets, partial: "projects/project",
      locals: { bucket: @bucket_pin.bucket } if @bucket_pin.moved_out_of_parent? %>
```

### Deleting a Stack

When destroying a Stack we want to close the modal and remove the stack Element. We’ll do that with Turbo Streams:

```
<%= turbo_stream.remove dom_id(@stack) %>
<%= turbo_stream.update :stack_modal, html: "" %>
```

[📺 Watch video](https://videos.37signals.com/dev/assets/videos/building-basecamp-project-stacks-with-hotwire/delete-stack.mp4)

### Opening and editing a Stack

For the rest of the CRUD we’ll use Turbo Frames out of the box.

```
<!-- projects/index.html.erb -->
<%= turbo_frame_tag :stack_modal %>

<!-- stacks/_stack.html.erb -->
<%= link_to stack_path(bucket), data: { turbo_frame: :stack_modal } %>
```

We’ll toggle the stack’s edit mode the same way:

```
<!-- stacks/show.html.erb -->
<%= turbo_frame_tag dom_id(@bucket.stack, :header) do %>
  <%= link_to @bucket.name, edit_stack_path(@bucket) %>
<% end %>

<!-- stacks/edit.html.erb -->
<%= turbo_frame_tag dom_id(@bucket.stack, :header) do %>
  <%= form_with model: @bucket.stack, url: stack_path(@bucket) do |form| %>
    <%= form.text_field :name, required: true, maxlength: 100 %>
    <%= form.submit "Save" %>
    <%= link_to "Never mind", stack_path(@bucket) %>
  <% end %>
<% end %>
```

[📺 Watch video](https://videos.37signals.com/dev/assets/videos/building-basecamp-project-stacks-with-hotwire/open-and-edit-stack-name.mp4)

---

## Conclusion

We haven’t covered everything that went into the stacks project but hopefully enough to demonstrate how a single programmer can efficiently take ownership of the entire tech stack using Hotwire.

With 37 lines of Javascript and a few lines of Turbo Stream Ruby code we added a rich and responsive UX on top of an existing legacy Javascript drag-and-drop sorting system without adding any complexity to it.
