# [Project Name]

[One-line description.]

## Stack

Ruby 4.0, Rails 8.1, SQLite 3, Hotwire (Turbo + Stimulus), vanilla CSS/JS, Importmap, Propshaft, Solid Trifecta (Queue/Cable/Cache), Kamal.

## Commands

```bash
bin/setup              # Initial setup
bin/dev                # Dev server
bin/rails test         # Unit tests
bin/rails test:system  # System tests
bin/ci                 # Full CI suite (run before handoff)
```

## Style

READ STYLE.md in this same directory. It defines Ruby/Rails code style (method ordering, visibility modifiers, conditionals, bang methods, CRUD controllers, model interactions, jobs).

## Rails Way — Non-Negotiable

References:
- https://guides.rubyonrails.org/ (Rails 8.1)
- https://dev.37signals.com/series/code-i-like/ (Jorge Manrubia)
- DHH, "On Writing Software Well" (episodes 1–5)

When in doubt, check the guides.

### Conventions over configuration

- **Generators first.** Always use `bin/rails g model|controller|migration|job|mailer`. Don't hand-write what Rails generates.
- **Migration version.** Always `ActiveRecord::Migration[8.1]` — never omit the version bracket.
- **Naming.** Models: singular (`User`). Controllers: plural (`UsersController`). Tables: plural (`users`). Foreign keys: `user_id`. Join tables: alphabetical (`groups_users`).

### Controllers

- **CRUD only.** 7 actions max: `index`, `show`, `new`, `create`, `edit`, `update`, `destroy`. No custom actions — extract a new resource instead.
- **Controllers access models directly.** Plain CRUD for simple cases, model methods for complex behavior. No intermediary layer.
- **Strong params.** Always `private` method, always `require().permit()`.
- **`form_with`** — never `form_for` or `form_tag` (removed in Rails 8).
- **Respond with conventions:** `redirect_to` after mutation, `render` on validation failure.

```ruby
# ✗ Custom action
resources :mail_accounts do
  post :verify
end

# ✓ New resource
resources :mail_accounts do
  resource :verification, only: :create
end

# ✓ Simple CRUD — plain ActiveRecord is fine
def create
  @mail_account = Current.user.mail_accounts.create!(mail_account_params)
end

# ✓ Complex behavior — call the model method
def create
  @bundle.deliver  # model hides the complexity
end
```

### Models — Vanilla Rails (Jorge Manrubia / 37signals)

Reference: https://dev.37signals.com/series/code-i-like/

- **Rich models.** ALL business logic lives in models. No service objects. No `app/services/`. No interactors. No use cases. Ever.
- **Vanilla Rails is plenty.** Controllers access domain models directly. No application layer between them.
- **Active Record, nice and blended.** Don't separate persistence from domain logic. Embrace it — that's the whole point of Active Record.
- **POROs belong in `app/models/` too.** Not everything needs to inherit from `ApplicationRecord`. Form objects, value objects, operation objects — they're all just domain models.
- **Domain driven boldness.** Name things after what they mean in the domain, not generic CRUD verbs. `person.decease` not `person.soft_delete`. `contact.designate_to(box)` not `ContactBoxAssigner.call`. Use a dictionary. Be expressive.

#### Concerns — the 37signals way

Two uses, different conventions:

1. **Shared concerns** across models → `app/models/concerns/` (`Taggable`, `Sluggable`)
2. **Model-specific concerns** → `app/models/<model>/` (`MailAccount::Collecting`, `Bundle::Delivering`)

Rules:
- Each concern must have genuine **"has trait" / "acts as"** semantics. Not arbitrary buckets.
- Concerns capture a **cohesive responsibility** — only things that belong together.
- The model file itself is mostly **declarations** (associations, validations, scopes, includes).
- For complex operations, the concern **delegates to additional classes** (good old OO). The model is a **facade** hiding a subsystem.

```ruby
# app/models/mail_account.rb — declarations only
class MailAccount < ApplicationRecord
  include Collecting, Verifiable

  belongs_to :user
  has_many :collected_messages, dependent: :destroy

  validates :address, presence: true
  scope :active, -> { where(active: true) }
end

# app/models/mail_account/collecting.rb — one responsibility
module MailAccount::Collecting
  extend ActiveSupport::Concern

  def collect_later
    MailAccount::CollectJob.perform_later(self)
  end

  def collect_now
    Collection.new(self).run  # delegates to a PORO when complex
  end
end

# app/models/mail_account/collection.rb — PORO for complex logic
class MailAccount::Collection
  def initialize(mail_account)
    @mail_account = mail_account
  end

  def run
    # connect, fetch, strip, store...
  end
end
```

**The API the caller sees is always the model:**
```ruby
# ✓ Natural, reads like Ruby
mail_account.collect_now
mail_account.verify

# ✗ Procedural, shifts composition burden to the caller
MailAccountCollectionService.new(mail_account).call
MailAccountVerifier.execute(mail_account)
```

#### Callbacks — embrace them (DHH ep.2)

Callbacks are **not a code smell** — they are how auxiliary complexity stays on the side. The main code path stays clean; side effects live in concerns via callbacks.

- **Use callbacks + jobs together:** callback decides if work is needed, then enqueues a job for the actual work. Don't block the request.
- **Two-step pattern:** `after_save` to inspect dirty attributes (inside transaction), `after_commit` to trigger the actual work (after transaction).
- **Suppression:** every callback system needs a clean opt-out for special cases (imports, copies, seeds).

```ruby
# Concern decides + defers
module Recording::Mentionable
  extend ActiveSupport::Concern

  included do
    after_save :remember_to_eavesdrop
    after_commit :eavesdrop_for_mentions
  end
end

# Suppression for special paths
Mention::Eavesdropper.suppressed { import_old_data }
```

#### Globals via Current (DHH ep.3)

`Current` is not evil — it's a sharp knife for request-scoped context. Keep it small (account, user, request details). Don't thread these through 5 layers of method params.

```ruby
# ✓ Use Current for request-scoped context
Current.user
Current.account

# ✗ Don't grow Current to 15+ attributes
```

#### POROs in app/models/ (DHH ep.4)

Not every model needs a database table. Form objects, value objects, operation encapsulations — they all live in `app/models/`. No `app/services/`, no `app/interactors/`.

**Pattern: concern as API gateway + composed POROs.** A thin concern provides a beautiful accessor on the host model, then delegates real work to plain Ruby classes composed together:

```ruby
# app/models/user/notifyee.rb — thin concern, just the doorway
module User::Notifyee
  extend ActiveSupport::Concern

  def notifications
    @notifications ||= Notifications.new(self)
  end
end

# app/models/notifications.rb — PORO, aggregates sub-concerns
class Notifications
  def initialize(user) = @user = user
  def granularity = Granularity.new(self)
  def schedule    = Schedule.new(self)
end

# Result: current.user.notifications.granularity.choice
# Instead of: NotificationGranularityService.new(current.user).call
```

#### Code comments are a smell (DHH ep.1)

If code needs a comment, extract an **explaining constant** or **explaining method** instead. Place constants near their usage (proximity > ordering).

```ruby
# ✗ Magic number with comment
# Wait 30 seconds for undo before destroying
remove_inaccessible_records_later(wait: 30.seconds)

# ✓ Explaining constant
GRACE_PERIOD_FOR_UNDO = 30.seconds
remove_inaccessible_records_later(wait: GRACE_PERIOD_FOR_UNDO)
```

#### Other model rules

- **Validations in models.** Never in controllers.
- **Scopes** for reusable queries. Class methods for complex queries.
- **Callbacks:** `after_create_commit`, `after_update_commit`, `after_destroy_commit` — avoid `after_save` when you need specificity. See "Callbacks" section above.
- **Encryption:** `encrypts :field` for sensitive data (Active Record Encryption).
- **No raw SQL.** Use ActiveRecord query interface. Arel when needed.

### Associations

- Always declare both sides (`has_many` + `belongs_to`).
- Use `dependent:` on every `has_many`/`has_one` — pick `destroy`, `delete_all`, `nullify`, or `restrict_with_error`.
- `has_many :through` over `has_and_belongs_to_many`.

### Migrations

- **One concern per migration.** Don't mix table creation with data manipulation.
- **Always reversible.** Use `change` (preferred) or `up`/`down`. Test rollback.
- **Column types matter:** `string` (short text), `text` (long), `integer`, `decimal` (money — never float), `boolean`, `datetime`, `references`.
- Add indexes on foreign keys and columns you query: `add_index` or `index: true` on `t.references`.
- **Never edit a migration that has been pushed.** Create a new one.

### Routes

- `resources` and `resource` — not manual `get`/`post` routes.
- Nest only 1 level deep. Shallow nesting when the child has its own identity.
- `only:` / `except:` to limit exposed actions.
- Singular `resource` for things that exist once per user (session, settings, profile).

### Jobs

- **Shallow jobs.** Job `perform` calls a model method — logic stays in the model.
- **`_later` / `_now` convention.** `model.do_thing_later` enqueues, `model.do_thing_now` executes.

```ruby
# In model concern
def collect_later
  MailAccount::CollectJob.perform_later(self)
end

def collect_now
  # actual logic here
end
```

### Views

- **Partials** for reuse. Prefix with `_`. Pass locals explicitly — no instance variables in partials.
- **Helpers** for view logic — not in templates, not in models.
- **Turbo Frames** for partial page updates. **Turbo Streams** for multi-target updates.
- **Stimulus** for JS behavior — small, focused controllers. No inline JS.

### Tests — test the real thing (DHH ep.5)

- **Minitest + fixtures** (Rails default). No RSpec, no FactoryBot. No mocks, no stubs.
- **Hit the real database.** Let callbacks run. Render real views. Test the actual thing, not a mock of it.
- **Fixtures as a shared world.** Pre-configured "characters in a world" you pull from — no 10-line factory setup per test. Specific objects for specific tests are created inline.
- **Multiple assertions per test is fine.** Test one *aspect*, not one assertion. 2–4 assertions per test is normal.
- **Two stock assertions:** `assert` and `assert_equal` cover almost everything. Extract domain-specific assertions only for complex repeated checks.
- **Speed: fast enough > blazing fast.** 0.5s/model test, 0.7s/controller test is acceptable. Don't distort production code to make tests faster.
- **No test damage.** Never change production code just to make it testable (injecting dependencies, adding interfaces for mocking). If the code works, test it as-is.
- **Controller tests = integration tests.** Full HTTP request → response → HTML assertion. Not narrow unit tests of controller methods.
- Test file mirrors app structure: `app/models/user.rb` → `test/models/user_test.rb`. Concern tests mirror too: `Recording::Lockable` → `test/models/recording/lock_test.rb`.
- System tests with Capybara for user flows (when JS behavior needs testing).
- **Bug → regression test first**, then fix.

### Authentication

- Rails 8 built-in: `bin/rails generate authentication`. Uses `has_secure_password`, `Session` model, `Current.user`.
- Don't roll your own. Don't add Devise.

### Frontend (Hotwire stack)

- **Importmap** — no Node, no bundler, no webpack/esbuild.
- **Propshaft** — no Sprockets.
  - **No `@import` in CSS.** Propshaft does not process CSS `@import` — they silently fail. No `application.css` manifest file needed (that's a Sprockets convention).
  - **`stylesheet_link_tag :app`** to bulk-load all CSS files from `app/assets/stylesheets/`. Each file gets its own `<link>` tag.
  - **`@layer` for ordering.** Declare layer order in `_global.css` (underscore sorts first). Each CSS file wraps its rules in `@layer base { }`, `@layer components { }`, etc.
- **No jQuery.** Stimulus handles behavior.
- **CSS:** vanilla CSS (or Dart Sass if configured). No Tailwind unless project specifies it.

## Ruby style

- **2-space indent.** No tabs.
- **Frozen string literals** — let Rubocop enforce it.
- **Double quotes** for strings (Rails convention).
- **Symbol array:** `%i[foo bar baz]`.
- **String array:** `%w[foo bar baz]`.
- **Hash shorthand (Ruby 4):** `{ x:, y: }` when value matches key name.
- **Pattern matching** where it improves clarity.
- **Endless methods** for trivial one-liners: `def name = @name`.

## Code

- No comments. If needed, use Rails notes (`bin/rails notes`):
  - `TODO:` / `FIXME:` / `OPTIMIZE:`
  - Format: `# TODO: initials DDmmmYY description`
- Files < 500 LOC; split if needed.

## Critical Thinking

- Fix root cause, not symptoms.
- Unsure → read more code; if still stuck → ask with short options.
- Conflicts → call out; pick safer path.
- Unrecognized changes → assume other agent; keep going; focus your changes.

## Git

- Conventional Commits (`feat|fix|refactor|build|ci|chore|docs|style|perf|test`).
- Small, reviewable diffs. One concern per commit.
- Safe by default — no destructive ops without consent.
- Never commit/push without explicit request.
- **Never push to main.** Branch + PR. Owner merges.

## CI

- Before handoff: run `bin/ci`. Fix until green.
- `gh run list/view` for CI status.

## Common AI Mistakes — Don't Do These

```
✗ form_for / form_tag           → ✓ form_with
✗ before_filter                 → ✓ before_action
✗ attr_accessible               → ✓ strong params (permit)
✗ render text: / render nothing → ✓ render plain: / head :ok
✗ find_by_id                    → ✓ find (raises) or find_by (nil)
✗ update_attributes             → ✓ update
✗ .where(id: x).first           → ✓ .find(x) or .find_by(id: x)
✗ Sprockets / Webpacker         → ✓ Propshaft + Importmap
✗ @import in CSS / application.css manifest → ✓ stylesheet_link_tag :app + @layer
✗ Devise / custom auth          → ✓ Rails 8 authentication generator
✗ FactoryBot / RSpec            → ✓ Fixtures + Minitest
✗ Service objects / app/services → ✓ Model concerns (ALWAYS)
✗ puts / p for debugging        → ✓ Rails.logger.debug
✗ ENV["X"] direct               → ✓ Rails.application.credentials or config
✗ raw SQL strings               → ✓ ActiveRecord query interface
✗ has_and_belongs_to_many       → ✓ has_many :through
✗ resources + custom actions    → ✓ resources + nested resource
✗ Inline JS / <script> tags    → ✓ Stimulus controllers
✗ jQuery / lodash               → ✓ Vanilla JS + Stimulus
✗ Migration without [8.1]       → ✓ ActiveRecord::Migration[8.1]
✗ Float for money               → ✓ decimal / integer (cents)
```
