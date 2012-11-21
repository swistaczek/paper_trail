<<<<<<< HEAD
# PaperTrail [![Build Status](https://secure.travis-ci.org/airblade/paper_trail.png)](http://travis-ci.org/airblade/paper_trail) [![Dependency Status](https://gemnasium.com/airblade/paper_trail.png)](https://gemnasium.com/airblade/paper_trail)

PaperTrail lets you track changes to your models' data.  It's good for auditing or versioning.  You can see how a model looked at any stage in its lifecycle, revert it to any version, and even undelete it after it's been destroyed.

There's an excellent [Railscast on implementing Undo with Paper Trail](http://railscasts.com/episodes/255-undo-with-paper-trail).


## Features

* Stores every create, update and destroy (or only the lifecycle events you specify).
* Does not store updates which don't change anything.
* Allows you to specify attributes (by inclusion or exclusion) which must change for a Version to be stored.
* Allows you to get at every version, including the original, even once destroyed.
* Allows you to get at every version even if the schema has since changed.
* Allows you to get at the version as of a particular time.
* Option to automatically restore `has_one` associations as they were at the time.
* Automatically records who was responsible via your controller.  PaperTrail calls `current_user` by default, if it exists, but you can have it call any method you like.
* Allows you to set who is responsible at model-level (useful for migrations).
* Allows you to store arbitrary model-level metadata with each version (useful for filtering versions).
* Allows you to store arbitrary controller-level information with each version, e.g. remote IP.
* Can be turned off/on per class (useful for migrations).
* Can be turned off/on per request (useful for testing with an external service).
* Can be turned off/on globally (useful for testing).
* No configuration necessary.
* Stores everything in a single database table by default (generates migration for you), or can use separate tables for separate models.
* Supports custom version classes so different models' versions can have different behaviour.
* Supports custom name for versions association.
* Thoroughly tested.
* Threadsafe.


## Rails Version

Works on Rails 3 and Rails 2.3.  The Rails 3 code is on the `master` branch and tagged `v2.x`.  The Rails 2.3 code is on the `rails2` branch and tagged `v1.x`.  Please note I'm not adding new features to the Rails 2.3 codebase.


## API Summary

When you declare `has_paper_trail` in your model, you get these methods:

    class Widget < ActiveRecord::Base
      has_paper_trail   # you can pass various options here
    end

    # Returns this widget's versions.  You can customise the name of the association.
    widget.versions

    # Return the version this widget was reified from, or nil if it is live.
    # You can customise the name of the method.
    widget.version

    # Returns true if this widget is the current, live one; or false if it is from a previous version.
    widget.live?

    # Returns who put the widget into its current state.
    widget.originator

    # Returns the widget (not a version) as it looked at the given timestamp.
    widget.version_at(timestamp)

    # Returns the widget (not a version) as it was most recently.
    widget.previous_version

    # Returns the widget (not a version) as it became next.
    widget.next_version

    # Turn PaperTrail off for all widgets.
    Widget.paper_trail_off

    # Turn PaperTrail on for all widgets.
    Widget.paper_trail_on

And a `Version` instance has these methods:

    # Returns the item restored from this version.
    version.reify(options = {})

    # Returns who put the item into the state stored in this version.
    version.originator

    # Returns who changed the item from the state it had in this version.
    version.terminator
    version.whodunnit

    # Returns the next version.
    version.next

    # Returns the previous version.
    version.previous

    # Returns the index of this version in all the versions.
    version.index

    # Returns the event that caused this version (create|update|destroy).
    version.event

In your controllers you can override these methods:

    # Returns the user who is responsible for any changes that occur.
    # Defaults to current_user.
    user_for_paper_trail

    # Returns any information about the controller or request that you want
    # PaperTrail to store alongside any changes that occur.
    info_for_paper_trail


## Basic Usage

PaperTrail is simple to use.  Just add 15 characters to a model to get a paper trail of every `create`, `update`, and `destroy`.

    class Widget < ActiveRecord::Base
      has_paper_trail
    end

This gives you a `versions` method which returns the paper trail of changes to your model.

    >> widget = Widget.find 42
    >> widget.versions             # [<Version>, <Version>, ...]

Once you have a version, you can find out what happened:

    >> v = widget.versions.last
    >> v.event                     # 'update' (or 'create' or 'destroy')
    >> v.whodunnit                 # '153'  (if the update was via a controller and
                                   #         the controller has a current_user method,
                                   #         here returning the id of the current user)
    >> v.created_at                # when the update occurred
    >> widget = v.reify            # the widget as it was before the update;
                                   # would be nil for a create event

PaperTrail stores the pre-change version of the model, unlike some other auditing/versioning plugins, so you can retrieve the original version.  This is useful when you start keeping a paper trail for models that already have records in the database.

    >> widget = Widget.find 153
    >> widget.name                                 # 'Doobly'

    # Add has_paper_trail to Widget model.

    >> widget.versions                             # []
    >> widget.update_attributes :name => 'Wotsit'
    >> widget.versions.first.reify.name            # 'Doobly'
    >> widget.versions.first.event                 # 'update'

This also means that PaperTrail does not waste space storing a version of the object as it currently stands.  The `versions` method gives you previous versions; to get the current one just call a finder on your `Widget` model as usual.

Here's a helpful table showing what PaperTrail stores:

<table>
  <tr>
    <th>Event</th>
    <th>Model Before</th>
    <th>Model After</th>
  </tr>
  <tr>
    <td>create</td>
    <td>nil</td>
    <td>widget</td>
  </tr>
  <tr>
    <td>update</td>
    <td>widget</td>
    <td>widget'</td>
  <tr>
    <td>destroy</td>
    <td>widget</td>
    <td>nil</td>
  </tr>
</table>

PaperTrail stores the values in the Model Before column.  Most other auditing/versioning plugins store the After column.


## Choosing Lifecycle Events To Monitor

You can choose which events to track with the `on` option.  For example, to ignore `create` events:

    class Article < ActiveRecord::Base
      has_paper_trail :on => [:update, :destroy]
    end


## Choosing When To Save New Versions

You can choose the conditions when to add new versions with the `if` and `unless` options. For example, to save versions only for US non-draft translations:

    class Translation < ActiveRecord::Base
      has_paper_trail :if     => Proc.new { |t| t.language_code == 'US' },
                      :unless => Proc.new { |t| t.type == 'DRAFT'       }
    end



## Choosing Attributes To Monitor

You can ignore changes to certain attributes like this:

    class Article < ActiveRecord::Base
      has_paper_trail :ignore => [:title, :rating]
    end

This means that changes to just the `title` or `rating` will not store another version of the article.  It does not mean that the `title` and `rating` attributes will be ignored if some other change causes a new `Version` to be created.  For example:

    >> a = Article.create
    >> a.versions.length                         # 1
    >> a.update_attributes :title => 'My Title', :rating => 3
    >> a.versions.length                         # 1
    >> a.update_attributes :content => 'Hello'
    >> a.versions.length                         # 2
    >> a.versions.last.reify.title               # 'My Title'

Or, you can specify a list of all attributes you care about:

    class Article < ActiveRecord::Base
      has_paper_trail :only => [:title]
    end

This means that only changes to the `title` will save a version of the article:

    >> a = Article.create
    >> a.versions.length                         # 1
    >> a.update_attributes :title => 'My Title'
    >> a.versions.length                         # 2
    >> a.update_attributes :content => 'Hello'
    >> a.versions.length                         # 2

Passing both `:ignore` and `:only` options will result in the article being saved if a changed attribute is included in `:only` but not in `:ignore`.

You can skip fields altogether with the `:skip` option.  As with `:ignore`, updates to these fields will not create a new `Version`.  In addition, these fields will not be included in the serialised version of the object whenever a new `Version` is created.

For example:

    class Article < ActiveRecord::Base
      has_paper_trail :skip => [:file_upload]
    end


## Reverting And Undeleting A Model

PaperTrail makes reverting to a previous version easy:

    >> widget = Widget.find 42
    >> widget.update_attributes :name => 'Blah blah'
    # Time passes....
    >> widget = widget.versions.last.reify  # the widget as it was before the update
    >> widget.save                          # reverted

Alternatively you can find the version at a given time:

    >> widget = widget.version_at(1.day.ago)  # the widget as it was one day ago
    >> widget.save                            # reverted

Note `version_at` gives you the object, not a version, so you don't need to call `reify`.

Undeleting is just as simple:

    >> widget = Widget.find 42
    >> widget.destroy
    # Time passes....
    >> widget = Version.find(153).reify    # the widget as it was before it was destroyed
    >> widget.save                         # the widget lives!

In fact you could use PaperTrail to implement an undo system, though I haven't had the opportunity yet to do it myself.  However [Ryan Bates has](http://railscasts.com/episodes/255-undo-with-paper-trail)!


## Navigating Versions

You can call `previous_version` and `next_version` on an item to get it as it was/became.  Note that these methods reify the item for you.

    >> widget = Widget.find 42
    >> widget.versions.length              # 4 for example
    >> widget = widget.previous_version    # => widget == widget.versions.last.reify
    >> widget = widget.previous_version    # => widget == widget.versions[-2].reify
    >> widget.next_version                 # => widget == widget.versions.last.reify
    >> widget.next_version                 # nil

As an aside, I'm undecided about whether `widget.versions.last.next_version` should return `nil` or `self` (i.e. `widget`).  Let me know if you have a view.

If instead you have a particular `version` of an item you can navigate to the previous and next versions.

    >> widget = Widget.find 42
    >> version = widget.versions[-2]    # assuming widget has several versions
    >> previous = version.previous
    >> next = version.next

You can find out which of an item's versions yours is:

    >> current_version_number = version.index    # 0-based

Finally, if you got an item by reifying one of its versions, you can navigate back to the version it came from:

    >> latest_version = Widget.find(42).versions.last
    >> widget = latest_version.reify
    >> widget.version == latest_version    # true

You can find out whether a model instance is the current, live one -- or whether it came instead from a previous version -- with `live?`:

    >> widget = Widget.find 42
    >> widget.live?                        # true
    >> widget = widget.versions.last.reify
    >> widget.live?                        # false


## Finding Out Who Was Responsible For A Change

If your `ApplicationController` has a `current_user` method, PaperTrail will store the value it returns in the `version`'s `whodunnit` column.  Note that this column is a string so you will have to convert it to an integer if it's an id and you want to look up the user later on:

    >> last_change = Widget.versions.last
    >> user_who_made_the_change = User.find last_change.whodunnit.to_i

You may want PaperTrail to call a different method to find out who is responsible.  To do so, override the `user_for_paper_trail` method in your controller like this:

    class ApplicationController
      def user_for_paper_trail
        logged_in? ? current_member : 'Public user'  # or whatever
      end
    end

In a migration or in `script/console` you can set who is responsible like this:

    >> PaperTrail.whodunnit = 'Andy Stewart'
    >> widget.update_attributes :name => 'Wibble'
    >> widget.versions.last.whodunnit              # Andy Stewart

N.B. A `version`'s `whodunnit` records who changed the object causing the `version` to be stored.  Because a `version` stores the object as it looked before the change (see the table above), `whodunnit` returns who stopped the object looking like this -- not who made it look like this.  Hence `whodunnit` is aliased as `terminator`.

To find out who made a `version`'s object look that way, use `version.originator`.  And to find out who made a "live" object look like it does, use `originator` on the object.

    >> widget = Widget.find 153                    # assume widget has 0 versions
    >> PaperTrail.whodunnit = 'Alice'
    >> widget.update_attributes :name => 'Yankee'
    >> widget.originator                           # 'Alice'
    >> PaperTrail.whodunnit = 'Bob'
    >> widget.update_attributes :name => 'Zulu'
    >> widget.originator                           # 'Bob'
    >> first_version, last_version = widget.versions.first, widget.versions.last
    >> first_version.whodunnit                     # 'Alice'
    >> first_version.originator                    # nil
    >> first_version.terminator                    # 'Alice'
    >> last_version.whodunnit                      # 'Bob'
    >> last_version.originator                     # 'Alice'
    >> last_version.terminator                     # 'Bob'


## Custom Version Classes

You can specify custom version subclasses with the `:class_name` option:

    class PostVersion < Version
      # custom behaviour, e.g:
      self.table_name = :post_versions
    end

    class Post < ActiveRecord::Base
      has_paper_trail :class_name => 'PostVersion'
    end

This allows you to store each model's versions in a separate table, which is useful if you have a lot of versions being created.

If you are using Postgres, you should also define the sequence that your custom version class will use:

    class PostVersion < Version
      self.table_name = :post_versions
      self.sequence_name = :post_version_id_seq
    end

Alternatively you could store certain metadata for one type of version, and other metadata for other versions.

If you only use custom version classes and don't use PaperTrail's built-in one, on Rails 3.2 you must:

- either declare PaperTrail's version class abstract like this (in `config/initializers/paper_trail_patch.rb`):

        Version.module_eval do
          self.abstract_class = true
        end

- or define a `versions` table in the database so Rails can instantiate the version superclass.

You can also specify custom names for the versions and version associations.  This is useful if you already have `versions` or/and `version` methods on your model.  For example:

    class Post < ActiveRecord::Base
      has_paper_trail :versions => :paper_trail_versions,
                      :version  => :paper_trail_version

      # Existing versions method.  We don't want to clash.
      def versions
        ...
      end
      # Existing version method.  We don't want to clash.
      def version
        ...
      end
    end


## Associations

I haven't yet found a good way to get PaperTrail to automatically restore associations when you reify a model.  See [here for a little more info](http://airbladesoftware.com/notes/undo-and-redo-with-papertrail).

If you can think of a good way to achieve this, please let me know.


## Has-One Associations

PaperTrail can restore `:has_one` associations as they were at (actually, 3 seconds before) the time.

    class Treasure < ActiveRecord::Base
      has_one :location
    end

    >> treasure.amount                  # 100
    >> treasure.location.latitude       # 12.345

    >> treasure.update_attributes :amount => 153
    >> treasure.location.update_attributes :latitude => 54.321

    >> t = treasure.versions.last.reify(:has_one => true)
    >> t.amount                         # 100
    >> t.location.latitude              # 12.345

The implementation is complicated by the edge case where the parent and child are updated in one go, e.g. in one web request or database transaction.  PaperTrail doesn't know about different models being updated "together", so you can't ask it definitively to get the child as it was before the joint parent-and-child update.

The correct solution is to make PaperTrail aware of requests or transactions (c.f. [Efficiency's transaction ID middleware](http://github.com/efficiency20/ops_middleware/blob/master/lib/e20/ops/middleware/transaction_id_middleware.rb)).  In the meantime we work around the problem by finding the child as it was a few seconds before the parent was updated.  By default we go 3 seconds before but you can change this by passing the desired number of seconds to the `:has_one` option:

    >> t = treasure.versions.last.reify(:has_one => 1)       # look back 1 second instead of 3

If you are shuddering, take solace from knowing PaperTrail opts out of these shenanigans by default. This means your `:has_one` associated objects will be the live ones, not the ones the user saw at the time.  Since PaperTrail doesn't auto-restore `:has_many` associations (I can't get it to work) or `:belongs_to` (I ran out of time looking at `:has_many`), this at least makes your associations wrong consistently ;)



## Has-Many-Through Associations

PaperTrail can track most changes to the join table.  Specifically it can track all additions but it can only track removals which fire the `after_destroy` callback on the join table.  Here are some examples:

Given these models:

    class Book < ActiveRecord::Base
      has_many :authorships, :dependent => :destroy
      has_many :authors, :through => :authorships, :source => :person
      has_paper_trail
    end

    class Authorship < ActiveRecord::Base
      belongs_to :book
      belongs_to :person
      has_paper_trail      # NOTE
    end

    class Person < ActiveRecord::Base
      has_many :authorships, :dependent => :destroy
      has_many :books, :through => :authorships
      has_paper_trail
    end

Then each of the following will store authorship versions:

    >> @book.authors << @dostoyevsky
    >> @book.authors.create :name => 'Tolstoy'
    >> @book.authorships.last.destroy
    >> @book.authorships.clear

But none of these will:

    >> @book.authors.delete @tolstoy
    >> @book.author_ids = [@solzhenistyn.id, @dostoyevsky.id]
    >> @book.authors = []

Having said that, you can apparently get all these working (I haven't tested it myself) with this patch:

    # In config/initializers/active_record_patch.rb
    module ActiveRecord
      # = Active Record Has Many Through Association
      module Associations
        class HasManyThroughAssociation < HasManyAssociation #:nodoc:
          alias_method :original_delete_records, :delete_records

          def delete_records(records, method)
            method ||= :destroy
            original_delete_records(records, method)
          end
        end
      end
    end

See [issue 113](https://github.com/airblade/paper_trail/issues/113) for a discussion about this.

There may be a way to store authorship versions, probably using association callbacks, no matter how the collection is manipulated but I haven't found it yet.  Let me know if you do.


## Storing metadata

You can store arbitrary model-level metadata alongside each version like this:

    class Article < ActiveRecord::Base
      belongs_to :author
      has_paper_trail :meta => { :author_id  => Proc.new { |article| article.author_id },
                                 :word_count => :count_words,
                                 :answer     => 42 }
      def count_words
        153
      end
    end

PaperTrail will call your proc with the current article and store the result in the `author_id` column of the `versions` table.

N.B.  You must also:

* Add your metadata columns to the `versions` table.
* Declare your metadata columns using `attr_accessible`.

For example:

    # config/initializers/paper_trail.rb
    class Version < ActiveRecord::Base
      attr_accessible :author_id, :word_count, :answer
    end

Why would you do this?  In this example, `author_id` is an attribute of `Article` and PaperTrail will store it anyway in serialized (YAML) form in the `object` column of the `version` record.  But let's say you wanted to pull out all versions for a particular author; without the metadata you would have to deserialize (reify) each `version` object to see if belonged to the author in question.  Clearly this is inefficient.  Using the metadata you can find just those versions you want:

    Version.all(:conditions => ['author_id = ?', author_id])

Note you can pass a symbol as a value in the `meta` hash to signal a method to call.

You can also store any information you like from your controller.  Just override the `info_for_paper_trail` method in your controller to return a hash whose keys correspond to columns in your `versions` table.  E.g.:

    class ApplicationController
      def info_for_paper_trail
        { :ip => request.remote_ip, :user_agent => request.user_agent }
      end
    end

Remember to add those extra columns to your `versions` table and use `attr_accessible` ;)


## Diffing Versions

There are two scenarios: diffing adjacent versions and diffing non-adjacent versions.

The best way to diff adjacent versions is to get PaperTrail to do it for you.  If you add an `object_changes` text column to your `versions` table, either at installation time with the `--with-changes` option or manually, PaperTrail will store the `changes` diff (excluding any attributes PaperTrail is ignoring) in each `update` version.  You can use the `version.changeset` method to retrieve it.  For example:

    >> widget = Widget.create :name => 'Bob'
    >> widget.versions.last.changeset                # {}
    >> widget.update_attributes :name => 'Robert'
    >> widget.versions.last.changeset                # {'name' => ['Bob', 'Robert']}

Note PaperTrail only stores the changes for updates; there's no point storing them for created or destroyed objects.

Please be aware that PaperTrail doesn't use diffs internally.  When I designed PaperTrail I wanted simplicity and robustness so I decided to make each version of an object self-contained.  A version stores all of its object's data, not a diff from the previous version.  This means you can delete any version without affecting any other.

To diff non-adjacent versions you'll have to write your own code.  These libraries may help:

For diffing two strings:

* [htmldiff](http://github.com/myobie/htmldiff): expects but doesn't require HTML input and produces HTML output.  Works very well but slows down significantly on large (e.g. 5,000 word) inputs.
* [differ](http://github.com/pvande/differ): expects plain text input and produces plain text/coloured/HTML/any output.  Can do character-wise, word-wise, line-wise, or arbitrary-boundary-string-wise diffs.  Works very well on non-HTML input.
* [diff-lcs](http://github.com/halostatue/ruwiki/tree/master/diff-lcs/trunk): old-school, line-wise diffs.

For diffing two ActiveRecord objects:

* [Jeremy Weiskotten's PaperTrail fork](http://github.com/jeremyw/paper_trail/blob/master/lib/paper_trail/has_paper_trail.rb#L151-156): uses ActiveSupport's diff to return an array of hashes of the changes.
* [activerecord-diff](http://github.com/tim/activerecord-diff): rather like ActiveRecord::Dirty but also allows you to specify which columns to compare.


## Turning PaperTrail Off/On

Sometimes you don't want to store changes.  Perhaps you are only interested in changes made by your users and don't need to store changes you make yourself in, say, a migration -- or when testing your application.

You can turn PaperTrail on or off in three ways: globally, per request, or per class.

### Globally

On a global level you can turn PaperTrail off like this:

    >> PaperTrail.enabled = false

For example, you might want to disable PaperTrail in your Rails application's test environment to speed up your tests.  This will do it:

    # in config/environments/test.rb
    config.after_initialize do
      PaperTrail.enabled = false
    end

If you disable PaperTrail in your test environment but want to enable it for specific tests, you can add a helper like this to your test helper:

    # in test/test_helper.rb
    def with_versioning
      was_enabled = PaperTrail.enabled?
      PaperTrail.enabled = true
      begin
        yield
      ensure
        PaperTrail.enabled = was_enabled
      end
    end

And then use it in your tests like this:

    test "something that needs versioning" do
      with_versioning do
        # your test
      end
    end

### Per request

You can turn PaperTrail on or off per request by adding a `paper_trail_enabled_for_controller` method to your controller which returns true or false:

    class ApplicationController < ActionController::Base
      def paper_trail_enabled_for_controller
        request.user_agent != 'Disable User-Agent'
      end
    end

### Per class

If you are about change some widgets and you don't want a paper trail of your changes, you can turn PaperTrail off like this:

    >> Widget.paper_trail_off

And on again like this:

    >> Widget.paper_trail_on

### Per method call

You can call a method without creating a new version using `without_versioning`.  It takes either a method name as a symbol:

    @widget.without_versioning :destroy

Or a block:

    @widget.without_versioning do
      @widget.update_attributes :name => 'Ford'
    end


## Deleting Old Versions

Over time your `versions` table will grow to an unwieldy size.  Because each version is self-contained (see the Diffing section above for more) you can simply delete any records you don't want any more.  For example:

    sql> delete from versions where created_at < 2010-06-01;

    >> Version.delete_all ["created_at < ?", 1.week.ago]


## Installation

### Rails 3

1. Install PaperTrail as a gem via your `Gemfile`:

    `gem 'paper_trail', '~> 2'`

2. Generate a migration which will add a `versions` table to your database.

    `bundle exec rails generate paper_trail:install`

3. Run the migration.

    `bundle exec rake db:migrate`

4. Add `has_paper_trail` to the models you want to track.

### Rails 2

Please see the `rails2` branch.


## Testing

PaperTrail uses Bundler to manage its dependencies (in development and testing).  You can run the tests with `bundle exec rake test`.  (You may need to `bundle install` first.)

It's a good idea to reset PaperTrail before each test so data from one test doesn't spill over another.  For example:

    RSpec.configure do |config|
      config.before :each do
        PaperTrail.controller_info = {}
        PaperTrail.whodunnit = nil
      end
    end

You may want to turn PaperTrail off to speed up your tests.  See the "Turning PaperTrail Off/On" section above.


## Articles

[Keep a Paper Trail with PaperTrail](http://www.linux-mag.com/id/7528), Linux Magazine, 16th September 2009.


## Problems

Please use GitHub's [issue tracker](http://github.com/airblade/paper_trail/issues).


## Contributors

Many thanks to:

* [Zachery Hostens](http://github.com/zacheryph)
* [Jeremy Weiskotten](http://github.com/jeremyw)
* [Phan Le](http://github.com/revo)
* [jdrucza](http://github.com/jdrucza)
* [conickal](http://github.com/conickal)
* [Thibaud Guillaume-Gentil](http://github.com/thibaudgg)
* Danny Trelogan
* [Mikl Kurkov](http://github.com/mkurkov)
* [Franco Catena](https://github.com/francocatena)
* [Emmanuel Gomez](https://github.com/emmanuel)
* [Matthew MacLeod](https://github.com/mattmacleod)
* [benzittlau](https://github.com/benzittlau)
* [Tom Derks](https://github.com/EgoH)
* [Jonas Hoglund](https://github.com/jhoglund)
* [Stefan Huber](https://github.com/MSNexploder)
* [thinkcast](https://github.com/thinkcast)
* [Dominik Sander](https://github.com/dsander)
* [Burke Libbey](https://github.com/burke)
* [6twenty](https://github.com/6twenty)
* [nir0](https://github.com/nir0)
* [Eduard Tsech](https://github.com/edtsech)
* [Mathieu Arnold](https://github.com/mat813)
* [Nicholas Thrower](https://github.com/throwern)
* [Benjamin Curtis](https://github.com/stympy)
* [Peter Harkins](https://github.com/pushcx)
* [Mohd Amree](https://github.com/amree)
* [Nikita Cernovs](https://github.com/nikitachernov)
* [Jason Noble](https://github.com/jasonnoble)
* [Jared Mehle](https://github.com/jrmehle)
* [Eric Schwartz](https://github.com/emschwar)
* [Ben Woosley](https://github.com/Empact)
* [Philip Arndt](https://github.com/parndt)
* [Daniel Vydra](https://github.com/dvydra)


## Inspirations

* [Simply Versioned](http://github.com/github/simply_versioned)
* [Acts As Audited](http://github.com/collectiveidea/acts_as_audited)


## Intellectual Property

Copyright (c) 2011 Andy Stewart (boss@airbladesoftware.com).
Released under the MIT licence.
=======
# Amoeba

Easy copying of rails associations such as `has_many`.

![amoebalogo](http://rocksolidwebdesign.com/wp_cms/wp-content/uploads/2012/02/amoeba_logo.jpg)

## What?

The goal was to be able to easily and quickly reproduce ActiveRecord objects including their children, for example copying a blog post maintaining its associated tags or categories.

This gem is named "Amoeba" because amoebas are (small life forms that are) good at reproducing. Their children and grandchildren also reproduce themselves quickly and easily.

### Technical Details

An ActiveRecord extension gem to allow the duplication of associated child record objects when duplicating an active record model. This gem overrides and adds to the built in `ActiveRecord::Base#dup` method.

Rails 3.2 compatible.

### Features

- Supports the following association types
    - `has_many`
    - `has_one :through`
    - `has_many :through`
    - `has_and_belongs_to_many`
- A simple DSL for configuration of which fields to copy. The DSL can be applied to your rails models or used on the fly.
- Supports STI (Single Table Inheritance) children inheriting their parent amoeba settings.
- Multiple configuration styles such as inclusive, exclusive and indiscriminate (aka copy everything).
- Supports cloning of the children of Many-to-Many records or merely maintaining original associations
- Supports automatic drill-down i.e. recursive copying of child and grandchild records.
- Supports preprocessing of fields to help indicate uniqueness and ensure the integrity of your data depending on your business logic needs, e.g. prepending "Copy of " or similar text.
- Supports preprocessing of fields with custom lambda blocks so you can do basically whatever you want if, for example, you need some custom logic while making copies.
- Amoeba can perform the following preprocessing operations on fields of copied records
    - set
    - prepend
    - append
    - nullify
    - customize
    - regex

## Usage

### Installation

is hopefully as you would expect:

    gem install amoeba

or just add it to your Gemfile:

    gem 'amoeba'

Configure your models with one of the styles below and then just run the `dup` method on your model as you normally would:

    p = Post.create(:title => "Hello World!", :content => "Lorum ipsum dolor")
    p.comments.create(:content => "I love it!")
    p.comments.create(:content => "This sucks!")

    puts Comment.all.count # should be 2

    my_copy = p.dup
    my_copy.save

    puts Comment.all.count # should be 4

By default, when enabled, amoeba will copy any and all associated child records automatically and associated them with the new parent record.

You can configure the behavior to only include fields that you list or to only include fields that you don't exclude. Of the three, the most performant will be the indiscriminate style, followed by the inclusive style, and the exclusive style will be the slowest because of the need for an extra explicit check on each field. This performance difference is likely negligible enough that you can choose the style to use based on which is easiest to read and write, however, if your data tree is large enough and you need control over what fields get copied, inclusive style is probably a better choice than exclusive style.

### Configuration

Please note that these examples are only loose approximations of real world scenarios and may not be particularly realistic, they are only for the purpose of demonstrating feature usage.

#### Indiscriminate Style

This is the most basic usage case and will simply enable the copying of any known associations.

If you have some models for a blog about like this:

    class Post < ActiveRecord::Base
      has_many :comments
    end

    class Comment < ActiveRecord::Base
      belongs_to :post
    end

simply add the amoeba configuration block to your model and call the enable method to enable the copying of child records, like this:

    class Post < ActiveRecord::Base
      has_many :comments

      amoeba do
        enable
      end
    end

    class Comment < ActiveRecord::Base
      belongs_to :post
    end

Child records will be automatically copied when you run the dup method.

#### Inclusive Style

If you only want some of the associations copied but not others, you may use the inclusive style:

    class Post < ActiveRecord::Base
      has_many :comments
      has_many :tags
      has_many :authors

      amoeba do
        enable
        include_field :tags
        include_field :authors
      end
    end

    class Comment < ActiveRecord::Base
      belongs_to :post
    end

Using the inclusive style within the amoeba block actually implies that you wish to enable amoeba, so there is no need to run the enable method, though it won't hurt either:

    class Post < ActiveRecord::Base
      has_many :comments
      has_many :tags
      has_many :authors

      amoeba do
        include_field :tags
        include_field :authors
      end
    end

    class Comment < ActiveRecord::Base
      belongs_to :post
    end

You may also specify fields to be copied by passing an array. If you call the `include_field` with a single value, it will be appended to the list of already included fields. If you pass an array, your array will overwrite the original values.

    class Post < ActiveRecord::Base
      has_many :comments
      has_many :tags
      has_many :authors

      amoeba do
        include_field [:tags, :authors]
      end
    end

    class Comment < ActiveRecord::Base
      belongs_to :post
    end

These examples will copy the post's tags and authors but not its comments.

The inclusive style, when used, will automatically disable any ther style that was previously selected.

#### Exclusive Style

If you have more fields to include than to exclude, you may wish to shorten the amount of typing and reading you need to do by using the exclusive style. All fields that are not explicitly excluded will be copied:

    class Post < ActiveRecord::Base
      has_many :comments
      has_many :tags
      has_many :authors

      amoeba do
        exclude_field :comments
      end
    end

    class Comment < ActiveRecord::Base
      belongs_to :post
    end

This example does the same thing as the inclusive style example, it will copy the post's tags and authors but not its comments. As with inclusive style, there is no need to explicitly enable amoeba when specifying fields to exclude.

The exclusive style, when used, will automatically disable any other style that was previously selected, so if you selected include fields, and then you choose some exclude fields, the `exclude_field` method will disable the previously slected inclusive style and wipe out any corresponding include fields.

#### Cloning

If you are using a Many-to-Many relationship, you may tell amoeba to actually make duplicates of the original related records rather than merely maintaining association with the original records. Cloning is easy, merely tell amoeba which fields to clone in the same way you tell it which fields to include or exclude.

    class Post < ActiveRecord::Base
      has_and_belongs_to_many :warnings

      has_many :post_widgets
      has_many :widgets, :through => :post_widgets

      amoeba do
        enable
        clone [:widgets, :tags]
      end
    end

    class Warning < ActiveRecord::Base
      has_and_belongs_to_many :posts
    end

    class PostWidget < ActiveRecord::Base
      belongs_to :widget
      belongs_to :post
    end

    class Widget < ActiveRecord::Base
      has_many :post_widgets
      has_many :posts, :through => :post_widgets
    end

This example will actually duplicate the warnings and widgets in the database. If there were originally 3 warnings in the database then, upon duplicating a post, you will end up with 6 warnings in the database. This is in contrast to the default behavior where your new post would merely be re-associated with any previously existing warnings and those warnings themselves would not be duplicated.

#### Limiting Association Types

By default, amoeba recognizes and attemps to copy any children of the following association types:

 - has one
 - has many
 - has and belongs to many

You may control which association types amoeba applies itself to by using the `recognize` method within the amoeba configuration block.

    class Post < ActiveRecord::Base
      has_one :config
      has_many :comments
      has_and_belongs_to_many :tags

      amoeba do
        recognize [:has_one, :has_and_belongs_to_many]
      end
    end

    class Comment < ActiveRecord::Base
      belongs_to :post
    end

    class Tag < ActiveRecord::Base
      has_and_belongs_to_many :posts
    end

This example will copy the post's configuration data and keep tags associated with the new post, but will not copy the post's comments because amoeba will only recognize and copy children of `has_one` and `has_and_belongs_to_many` associations and in this example, comments are not an `has_and_belongs_to_many` association.

### Field Preprocessors

#### Nullify

If you wish to prevent a regular (non `has_*` association based) field from retaining it's value when copied, you may "zero out" or "nullify" the field, like this:

    class Topic < ActiveRecord::Base
      has_many :posts
    end

    class Post < ActiveRecord::Base
      belongs_to :topic
      has_many :comments

      amoeba do
        enable
        nullify :date_published
        nullify :topic_id
      end
    end

    class Comment < ActiveRecord::Base
      belongs_to :post
    end

This example will copy all of a post's comments. It will also nullify the publishing date and dissociate the post from its original topic.

Unlike inclusive and exclusive styles, specifying null fields will not automatically enable amoeba to copy all child records. As with any active record object, the default field value will be used instead of `nil` if a default value exists on the migration.

#### Set

If you wish to just set a field to an aribrary value on all duplicated objects you may use the `set` directive. For example, if you wanted to copy an object that has some kind of approval process associated with it, you likely may wish to set the new object's state to be open or "in progress" again.

    class Post < ActiveRecord::Base
      amoeba do
        set :state_tracker => "open_for_editing"
      end
    end

In this example, when a post is duplicated, it's `state_tracker` field will always be given a value of `open_for_editing` to start.

#### Prepend

You may add a string to the beginning of a copied object's field during the copy phase:

    class Post < ActiveRecord::Base
      amoeba do
        enable
        prepend :title => "Copy of "
      end
    end

#### Append

You may add a string to the end of a copied object's field during the copy phase:

    class Post < ActiveRecord::Base
      amoeba do
        enable
        append :title => "Copy of "
      end
    end

#### Regex

You may run a search and replace query on a copied object's field during the copy phase:

    class Post < ActiveRecord::Base
      amoeba do
        enable
        regex :contents => {:replace => /dog/, :with => 'cat'}
      end
    end

#### Custom Methods

##### Customize

You may run a custom method or methods to do basically anything you like, simply pass a lambda block, or an array of lambda blocks to the `customize` directive. Each block must have the same form, meaning that each block must accept two parameters, the original object and the newly copied object. You may then do whatever you wish, like this:

    class Post < ActiveRecord::Base
      amoeba do
        prepend :title => "Hello world! "

        customize(lambda { |original_post,new_post|
          if original_post.foo == "bar"
            new_post.baz = "qux"
          end
        })

        append :comments => "... know what I'm sayin?"
      end
    end

or this, using an array:

    class Post < ActiveRecord::Base
      has_and_belongs_to_many :tags

      amoeba do
        include_field :tags

        customize([
          lambda do |orig_obj,copy_of_obj|
            # good stuff goes here
          end,

          lambda do |orig_obj,copy_of_obj|
            # more good stuff goes here
          end
        ])
      end
    end

##### Override

Lambda blocks passed to customize run, by default, after all copying and field pre-processing. If you wish to run a method before any customization or field pre-processing, you may use `override` the cousin of `customize`. Usage is the same as above.

    class Post < ActiveRecord::Base
      amoeba do
        prepend :title => "Hello world! "

        override(lambda { |original_post,new_post|
          if original_post.foo == "bar"
            new_post.baz = "qux"
          end
        })

        append :comments => "... know what I'm sayin?"
      end
    end

#### Chaining

You may apply a single preprocessor to multiple fields at once.

    class Post < ActiveRecord::Base
      amoeba do
        enable
        prepend :title => "Copy of ", :contents => "Copied contents: "
      end
    end

#### Stacking

You may apply multiple preproccessing directives to a single model at once.

    class Post < ActiveRecord::Base
      amoeba do
        prepend :title => "Copy of ", :contents => "Original contents: "
        append :contents => " (copied version)"
        regex :contents => {:replace => /dog/, :with => 'cat'}
      end
    end

This example should result in something like this:

    post = Post.create(
      :title => "Hello world",
      :contents =>  "I like dogs, dogs are awesome."
    )

    new_post = post.dup

    new_post.title # "Copy of Hello world"
    new_post.contents # "Original contents: I like cats, cats are awesome. (copied version)"

Like `nullify`, the preprocessing directives do not automatically enable the copying of associated child records. If only preprocessing directives are used and you do want to copy child records and no `include_field` or `exclude_field` list is provided, you must still explicitly enable the copying of child records by calling the enable method from within the amoeba block on your model.

### Precedence

You may use a combination of configuration methods within each model's amoeba block. Recognized association types take precedence over inclusion or exclusion lists. Inclusive style takes precedence over exclusive style, and these two explicit styles take precedence over the indiscriminate style. In other words, if you list fields to copy, amoeba will only copy the fields you list, or only copy the fields you don't exclude as the case may be. Additionally, if a field type is not recognized it will not be copied, regardless of whether it appears in an inclusion list. If you want amoeba to automatically copy all of your child records, do not list any fields using either `include_field` or `exclude_field`.

The following example syntax is perfectly valid, and will result in the usage of inclusive style. The order in which you call the configuration methods within the amoeba block does not matter:

    class Topic < ActiveRecord::Base
      has_many :posts
    end

    class Post < ActiveRecord::Base
      belongs_to :topic
      has_many :comments
      has_many :tags
      has_many :authors

      amoeba do
        exclude_field :authors
        include_field :tags
        nullify :date_published
        prepend :title => "Copy of "
        append :contents => " (copied version)"
        regex :contents => {:replace => /dog/, :with => 'cat'}
        include_field :authors
        enable
        nullify :topic_id
      end
    end

    class Comment < ActiveRecord::Base
      belongs_to :post
    end

This example will copy all of a post's tags and authors, but not its comments. It will also nullify the publishing date and dissociate the post from its original topic. It will also preprocess the post's fields as in the previous preprocessing example.

Note that, because of precedence, inclusive style is used and the list of exclude fields is never consulted. Additionally, the `enable` method is redundant because amoeba is automatically enabled when using `include_field`.

The preprocessing directives are run after child records are copied andare run in this order.

 1. Null fields
 2. Prepends
 3. Appends
 4. Search and Replace

Preprocessing directives do not affect inclusion and exclusion lists.

### Recursing

You may cause amoeba to keep copying down the chain as far as you like, simply add amoeba blocks to each model you wish to have copy its children. Amoeba will automatically recurse into any enabled grandchildren and copy them as well.

    class Post < ActiveRecord::Base
      has_many :comments

      amoeba do
        enable
      end
    end

    class Comment < ActiveRecord::Base
      belongs_to :post
      has_many :ratings

      amoeba do
        enable
      end
    end

    class Rating < ActiveRecord::Base
      belongs_to :comment
    end

In this example, when a post is copied, amoeba will copy each all of a post's comments and will also copy each comment's ratings.

### Has One Through

Using the `has_one :through` association is simple, just be sure to enable amoeba on the each model with a `has_one` association and amoeba will automatically and recursively drill down, like so:

    class Supplier < ActiveRecord::Base
      has_one :account
      has_one :history, :through => :account

      amoeba do
        enable
      end
    end

    class Account < ActiveRecord::Base
      belongs_to :supplier
      has_one :history

      amoeba do
        enable
      end
    end

    class History < ActiveRecord::Base
      belongs_to :account
    end

### Has Many Through

Copying of `has_many :through` associations works automatically. They perform the copy in the same way as the `has_and_belongs_to_many` association, meaning the actual child records are not copied, but rather the associations are simply maintained. You can add some field preprocessors to the middle model if you like but this is not strictly necessary:

    class Assembly < ActiveRecord::Base
      has_many :manifests
      has_many :parts, :through => :manifests

      amoeba do
        enable
      end
    end

    class Manifest < ActiveRecord::Base
      belongs_to :assembly
      belongs_to :part

      amoeba do
        prepend :notes => "Copy of "
      end
    end

    class Part < ActiveRecord::Base
      has_many :manifests
      has_many :assemblies, :through => :manifests

      amoeba do
        enable
      end
    end

### On The Fly Configuration

You may control how amoeba copies your object, on the fly, by passing a configuration block to the model's amoeba method. The configuration method is static but the configuration is applied on a per instance basis.

    class Post < ActiveRecord::Base
      has_many :comments

      amoeba do
        enable
        prepend :title => "Copy of "
      end
    end

    class Comment < ActiveRecord::Base
      belongs_to :post
    end

    class PostsController < ActionController
      def duplicate_a_post
        old_post = Post.create(
          :title => "Hello world",
          :contents => "Lorum ipsum"
        )

        old_post.class.amoeba do
          prepend :contents => "Here's a copy: "
        end

        new_post = old_post.dup

        new_post.title # should be "Copy of Hello world"
        new_post.contents # should be "Here's a copy: Lorum ipsum"
        new_post.save
      end
    end

### Inheritance

If you are using the Single Table Inheritance provided by ActiveRecord, you may cause amoeba to automatically process child classes in the same way as their parents. All you need to do is call the `propagate` method within the amoeba block of the parent class and all child classes should copy in a similar manner.

    create_table :products, :force => true do |t|
      t.string :type # this is the STI column

      # these belong to all products
      t.string :title
      t.decimal :price

      # these are for shirts only
      t.decimal :sleeve_length
      t.decimal :collar_size

      # these are for computers only
      t.integer :ram_size
      t.integer :hard_drive_size
    end

    class Product < ActiveRecord::Base
      has_many :images
      has_and_belongs_to_many :categories

      amoeba do
        enable
        propagate
      end
    end

    class Shirt < Product
    end

    class Computer < Product
    end

    class ProductsController
      def some_method
        my_shirt = Shirt.find(1)
        my_shirt.dup
        my_shirt.save

        # this shirt should now:
        # - have its own copy of all parent images
        # - be in the same categories as the parent
      end
    end

This example should duplicate all the images and sections associated with this Shirt, which is a child of Product

#### Parenting Style

By default, propagation uses submissive parenting, meaning the config settings on the parent will be applied, but any child settings, if present, will either add to or overwrite the parent settings depending on how you call the DSL methods.

You may change this behavior, the so called "parenting style", to give preference to the parent settings or to ignore any and all child settings.

##### Relaxed Parenting

The `:relaxed` parenting style will prefer parent settings.

    class Product < ActiveRecord::Base
      has_many :images
      has_and_belongs_to_many :sections

      amoeba do
        exclude_field :images
        propagate :relaxed
      end
    end

    class Shirt < Product
      include_field :images
      include_field :sections
      prepend :title => "Copy of "
    end

In this example, the conflicting `include_field` settings on the child will be ignored and the parent `exclude_field` setting will be used, while the `prepend` setting on the child will be honored because it doesn't conflict with the parent.

##### Strict Parenting

The `:strict` style will ignore child settings altogether and inherit any parent settings.

    class Product < ActiveRecord::Base
      has_many :images
      has_and_belongs_to_many :sections

      amoeba do
        exclude_field :images
        propagate :strict
      end
    end

    class Shirt < Product
      include_field :images
      include_field :sections
      prepend :title => "Copy of "
    end

In this example, the only processing that will happen when a Shirt is duplicated is whatever processing is allowed by the parent. So in this case the parent's `exclude_field` directive takes precedence over the child's `include_field` settings, and not only that, but none of the other settings for the child are used either. The `prepend` setting of the child is completely ignored.

##### Parenting and Precedence

Because of the two general forms of DSL config parameter usage, you may wish to make yourself mindful of how your coding style will affect the outcome of duplicating an object.

Just remember that:

* If you pass an array you will wipe all previous settings
* If you pass single values, you will add to currently existing settings

This means that, for example:

* When using the submissive parenting style, you can child take full precedence on a per field basis by passing an array of config values. This will cause the setting from the parent to be overridden instead of added to.
* When using the relaxed parenting style, you can still let the parent take precedence on a per field basis by passing an array of config values. This will cause the setting for that child to be overridden instead of added to.

##### A Submissive Override Example

This version will use both the parent and child settings, so both the images and sections will be copied.

    class Product < ActiveRecord::Base
      has_many :images
      has_and_belongs_to_many :sections

      amoeba do
        include_field :images
        propagate
      end
    end

    class Shirt < Product
      include_field :sections
    end

The next version will use only the child settings because passing an array will override any previous settings rather than adding to them and the child config takes precedence in the `submissive` parenting style. So in this case only the sections will be copied.

    class Product < ActiveRecord::Base
      has_many :images
      has_and_belongs_to_many :sections

      amoeba do
        include_field :images
        propagate
      end
    end

    class Shirt < Product
      include_field [:sections]
    end

##### A Relaxed Override Example

This version will use both the parent and child settings, so both the images and sections will be copied.

    class Product < ActiveRecord::Base
      has_many :images
      has_and_belongs_to_many :sections

      amoeba do
        include_field :images
        propagate :relaxed
      end
    end

    class Shirt < Product
      include_field :sections
    end

The next version will use only the parent settings because passing an array will override any previous settings rather than adding to them and the parent config takes precedence in the `relaxed` parenting style. So in this case only the images will be copied.

    class Product < ActiveRecord::Base
      has_many :images
      has_and_belongs_to_many :sections

      amoeba do
        include_field [:images]
        propagate
      end
    end

    class Shirt < Product
      include_field :sections
    end

### Validating Nested Attributes

If you end up with some validation issues when trying to validate the presence of a child's `belongs_to` association, just be sure to include the `:inverse_of` declaration on your relationships and all should be well.

For example this will throw a validation error saying that your posts are invalid:

    class Author < ActiveRecord::Base
      has_many :posts

      amoeba do
        enable
      end
    end

    class Post < ActiveRecord::Base
      belongs_to :author
      validates_presence_of :author

      amoeba do
        enable
      end
    end

    author = Author.find(1)
    author.dup

    author.save # this will fail validation

Wheras this will work fine:

    class Author < ActiveRecord::Base
      has_many :posts, :inverse_of => :author

      amoeba do
        enable
      end
    end

    class Post < ActiveRecord::Base
      belongs_to :author, :inverse_of => :posts
      validates_presence_of :author

      amoeba do
        enable
      end
    end

    author = Author.find(1)
    author.dup

    author.save # this will pass validation

This issue is not amoeba specific and also occurs when creating new objects using `accepts_nested_attributes_for`, like this:

    class Author < ActiveRecord::Base
      has_many :posts
      accepts_nested_attributes_for :posts
    end

    class Post < ActiveRecord::Base
      belongs_to :author
      validates_presence_of :author
    end

    # this will fail validation
    author = Author.create({:name => "Jim Smith", :posts => [{:title => "Hello World", :contents => "Lorum ipsum dolor}]})

This issue with `accepts_nested_attributes_for` can also be solved by using `:inverse_of`, like this:

    class Author < ActiveRecord::Base
      has_many :posts, :inverse_of => :author
      accepts_nested_attributes_for :posts
    end

    class Post < ActiveRecord::Base
      belongs_to :author, :inverse_of => :posts
      validates_presence_of :author
    end

    # this will pass validation
    author = Author.create({:name => "Jim Smith", :posts => [{:title => "Hello World", :contents => "Lorum ipsum dolor}]})

The crux of the issue is that upon duplication, the new `Author` instance does not yet have an ID because it has not yet been persisted, so the `:posts` do not yet have an `:author_id` either, and thus no `:author` and thus they will fail validation. This issue may likely affect amoeba usage so if you get some validation failures, be sure to add `:inverse_of` to your models.

## Configuration Reference

Here is a static reference to the available configuration methods, usable within the amoeba block on your rails models.

### Controlling Associations

#### enable

  Enables amoeba in the default style of copying all known associated child records. Using the enable method is only required if you wish to enable amoeba but you are not using either the `include_field` or `exclude_field` directives. If you use either inclusive or exclusive style, amoeba is automatically enabled for you, so calling `enable` would be redundant, though it won't hurt.

#### include_field

  Adds a field to the list of fields which should be copied. All associations not in this list will not be copied. This method may be called multiple times, once per desired field, or you may pass an array of field names. Passing a single symbol will add to the list of included fields. Passing an array will empty the list and replace it with the array you pass.

#### exclude_field

  Adds a field to the list of fields which should not be copied. Only the associations that are not in this list will be copied. This method may be called multiple times, once per desired field, or you may pass an array of field names. Passing a single symbol will add to the list of excluded fields. Passing an array will empty the list and replace it with the array you pass.

#### clone

  Adds a field to the list of associations which should have their associated children actually cloned. This means for example, that instead of just maintaining original associations with previously existing tags, a copy will be made of each tag, and the new record will be associated with these new tag copies rather than the old tag copies. This method may be called multiple times, once per desired field, or you may pass an array of field names. Passing a single symbol will add to the list of excluded fields. Passing an array will empty the list and replace it with the array you pass.

#### propagate

  This causes any inherited child models to take the same config settings when copied. This method may take up to one argument to control the so called "parenting style". The argument should be one of `strict`, `relaxed` or `submissive`.

  The default "parenting style" is `submissive`

  for example

        amoeba do
          propagate :strict
        end

  will choose the strict parenting style of inherited settings.

#### raised

  This causes any child to behave with a (potentially) different "parenting style" than its actual parent. This method takes up to a single parameter for which there are three options, `strict`, `relaxed` and `submissive`.

  The default "parenting style" is `submissive`

  for example:

        amoeba do
          raised :relaxed
        end

  will choose the relaxed parenting style of inherited settings for this child. A parenting style set via the `raised` method takes precedence over the parenting style set using the `propagate` method.

### Pre-Processing Fields

#### nullify

  Adds a field to the list of non-association based fields which should be set to nil during copy. All fields in this list will be set to `nil` - note that any nullified field will be given its default value if a default value exists on this model's migration. This method may be called multiple times, once per desired field, or you may pass an array of field names. Passing a single symbol will add to the list of null fields. Passing an array will empty the list and replace it with the array you pass.

#### prepend

  Prefix a field with some text. This only works for string fields. Accepts a hash of fields to prepend. The keys are the field names and the values are the prefix strings. An example scenario would be to add a string such as "Copy of " to your title field. Don't forget to include extra space to the right if you want it. Passing a hash will add each key value pair to the list of prepend directives. If you wish to empty the list of directives, you may pass the hash inside of an array like this `[{:title => "Copy of "}]`.

#### append

  Append some text to a field. This only works for string fields. Accepts a hash of fields to prepend. The keys are the field names and the values are the prefix strings. An example would be to add " (copied version)" to your description field. Don't forget to add a leading space if you want it. Passing a hash will add each key value pair to the list of append directives. If you wish to empty the list of directives, you may pass the hash inside of an array like this `[{:contents => " (copied version)"}]`.

#### set

  Set a field to a given value. This sould work for almost any type of field. Accepts a hash of fields and the values you want them set to.. The keys are the field names and the values are the prefix strings. An example would be to add " (copied version)" to your description field. Don't forget to add a leading space if you want it. Passing a hash will add each key value pair to the list of append directives. If you wish to empty the list of directives, you may pass the hash inside of an array like this `[{:approval_state => "open_for_editing"}]`.

#### regex

  Globally search and replace the field for a given pattern. Accepts a hash of fields to run search and replace upon. The keys are the field names and the values are each a hash with information about what to find and what to replace it with. in the form of . An example would be to replace all occurrences of the word "dog" with the word "cat", the parameter hash would look like this `:contents => {:replace => /dog/, :with => "cat"}`. Passing a hash will add each key value pair to the list of regex directives. If you wish to empty the list of directives, you may pass the hash inside of an array like this `[{:contents => {:replace => /dog/, :with => "cat"}]`.

#### override

  Runs a custom method so you can do basically whatever you want. All you need to do is pass a lambda block or an array of lambda blocks that take two parameters, the original object and the new object copy. These blocks will run before any other duplication or field processing.

  This method may be called multiple times, once per desired customizer block, or you may pass an array of lambdas. Passing a single lambda will add to the list of processing directives. Passing an array will empty the list and replace it with the array you pass.

#### customize

  Runs a custom method so you can do basically whatever you want. All you need to do is pass a lambda block or an array of lambda blocks that take two parameters, the original object and the new object copy. These blocks will run after all copying and field processing.

  This method may be called multiple times, once per desired customizer block, or you may pass an array of lambdas. Passing a single lambda will add to the list of processing directives. Passing an array will empty the list and replace it with the array you pass.

## Known Limitations and Issues

The regular expression preprocessor uses case-sensitive `String#gsub`. Given the performance decreases inherrent in using regular expressions already, the fact that character classes can essentially account for case-insensitive searches, the desire to keep the DSL simple and the general use cases for this gem, I don't see a good reason to add yet more decision based conditional syntax to accommodate using case-insensitive searches or singular replacements with `String#sub`. If you find yourself wanting either of these features, by all means fork the code base and if you like your changes, submit a pull request.

The behavior when copying nested hierarchical models is undefined. Copying a category model which has a `parent_id` field pointing to the parent category, for example, is currently undefined.

The behavior when copying polymorphic `has_many` associations is also undefined. Support for these types of associations is planned for a future release.

## For Developers

You may run the rspec tests like this:

    bundle exec rspec spec

### TODO

* add ability to cancel further processing from within an override block
* write some spec for the override method

## License

[The BSD License](http://www.opensource.org/licenses/bsd-license.php)

Copyright (c) 2012, Vaughn Draughon
All rights reserved.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

 - Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.
 - Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
>>>>>>> adcf423075c29aaa4a1f100f8ea041143e21beb6
