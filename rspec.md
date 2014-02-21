## RSpec

* Use just one expectation per example.

    ```Ruby
    # bad
    describe ArticlesController do
      #...

      describe 'GET new' do
        it 'assigns new article and renders the new article template' do
          get :new
          assigns[:article].should be_a_new Article
          response.should render_template :new
        end
      end

      # ...
    end

    # good
    describe ArticlesController do
      #...

      describe 'GET new' do
        it 'assigns a new article' do
          get :new
          assigns[:article].should be_a_new Article
        end

        it 'renders the new article template' do
          get :new
          response.should render_template :new
        end
      end

    end
    ```

* Make heavy use of `describe` and `context`
* Name the `describe` blocks as follows:
  * use "description" for non-methods
  * use pound "#method" for instance methods
  * use dot ".method" for class methods

    ```Ruby
    class Article
      def summary
        #...
      end

      def self.latest
        #...
      end
    end

    # the spec...
    describe Article do
      describe '#summary' do
        #...
      end

      describe '.latest' do
        #...
      end
    end
    ```

* Use [FactoryGirl](https://github.com/thoughtbot/factory_girl/blob/master/GETTING_STARTED.md#using-factories) to create test objects.

    ```Ruby
    # instantiating a model
    article = create(:article)
    ``` 

* Make heavy use of mocks and stubs

    ```Ruby
    # mocking a model
    article = mock_model(Article)

    # stubbing a method
    Article.stub(:find).with(article.id).and_return(article)
    ```

* When mocking a model, use the `as_null_object` method. It tells the
  output to listen only for messages we expect and ignore any other
  messages.

    ```Ruby
    article = mock_model(Article).as_null_object
    ```

* Use `let` blocks instead of `before(:each)` blocks with instance variables 
  to create data for the spec examples. `let` blocks get lazily evaluated.

    ```Ruby
    # use this:
    let(:article) { create(:article) }

    # ... instead of this:
    before(:each) { @article = create(:article) }
    ```

* Use `shared_examples` if you want to create a spec group that can be shared by many other tests.

   ```Ruby
   # bad
    describe Array do
      let(:numbers) { Array.new [7, 2, 4] }

      context "initialized with 3 items" do
        it "has a size of 3" do
          numbers.size.should eq(3)
        end
      end
    end

    describe Set do
      let(:numbers) { Set.new [7, 2, 4] }

      context "initialized with 3 items" do
        it "has a size of 3" do
          numbers.size.should eq(3)
        end
      end
    end

   #good
    shared_examples "a collection" do
      let(:numbers) { described_class.new([7, 2, 4]) }

      context "initialized with 3 items" do
        it "has a size of 3" do
          numbers.size.should eq(3)
        end
      end
    end

    describe Array do
      it_behaves_like "a collection"
    end

    describe Set do
      it_behaves_like "a collection"
    end
    ```

### Views

* The directory structure of the view specs `spec/unit/views` matches the
  one in `app/views`. For example the specs for the views in
  `app/views/users` are placed in `spec/unit/views/users`.
* The naming convention for the view specs is adding `_spec.rb` to the
  view name, for example the view `_form.html.haml` has a
  corresponding spec `_form.html.haml_spec.rb`.
* `spec_helper.rb` needs to be required in each view spec file.
* The outer `describe` block uses the path to the view without the
  `app/views` part. This is used by the `render` method when it is
  called without arguments.

    ```Ruby
    # spec/unit/views/articles/new.html.haml_spec.rb
    require 'spec_helper'

    describe 'articles/new.html.haml' do
      # ...
    end
    ```

* Always mock the models in the view specs. The purpose of the view is
  only to display information.
* The method `assign` supplies the instance variables which the view
  uses and are supplied by the controller.

    ```Ruby
    # spec/unit/views/articles/edit.html.haml_spec.rb
    describe 'articles/edit.html.haml' do
    it 'renders the form for a new article creation' do
      assign(
        :article,
        mock_model(Article).as_new_record.as_null_object
      )
      render
      rendered.should have_selector('form',
        method: 'post',
        action: articles_path
      ) do |form|
        form.should have_selector('input', type: 'submit')
      end
    end
    ```

* When a view uses helper methods, these methods need to be
  stubbed. Stubbing the helper methods is done on the `template`
  object:

    ```Ruby
    # app/helpers/articles_helper.rb
    class ArticlesHelper
      def formatted_date(date)
        # ...
      end
    end

    # app/views/articles/show.html.haml
    = "Published at: #{formatted_date(@article.published_at)}"

    # spec/unit/views/articles/show.html.haml_spec.rb
    describe 'articles/show.html.haml' do
      it 'displays the formatted date of article publishing' do
        article = mock_model(Article, published_at: Date.new(2012, 01, 01))
        assign(:article, article)

        template.stub(:formatted_date).with(article.published_at).and_return('01.01.2012')

        render
        rendered.should have_content('Published at: 01.01.2012')
      end
    end
    ```

* The helpers specs are separated from the view specs in the `spec/unit/helpers` directory.

### Controllers

* The directory structure of the controller specs in `spec/unit/controllers` matches the
  one in `app/controllers`. For example the specs for the controller in
  `app/controllers/admin/users_controller.rb` are placed in `spec/unit/controllers/admin/users_controller_spec.rb`.
* Mock the models and stub their methods. Testing the controller should not depend on the model creation.
* Test only the behaviour the controller should be responsible about:
  * Execution of particular methods
  * Data returned from the action - assigns, etc.
  * Result from the action - template render, redirect, etc.

        ```Ruby
        # Example of a commonly used controller spec
        # spec/unit/controllers/articles_controller_spec.rb
        # We are interested only in the actions the controller should perform
        # So we are mocking the model creation and stubbing its methods
        # And we concentrate only on the things the controller should do

        describe ArticlesController do
          # The model will be used in the specs for all methods of the controller
          let(:article) { mock_model(Article) }

          describe 'POST create' do
            before { Article.stub(:new).and_return(article) }

            it 'creates a new article with the given attributes' do
              Article.should_receive(:new).with(title: 'The New Article Title').and_return(article)
              post :create, message: { title: 'The New Article Title' }
            end

            it 'saves the article' do
              article.should_receive(:save)
              post :create
            end

            it 'redirects to the Articles index' do
              article.stub(:save)
              post :create
              response.should redirect_to(action: 'index')
            end
          end
        end
        ```

* Use context when the controller action has different behaviour depending on the received params.

    ```Ruby
    # A classic example for use of contexts in a controller spec is creation or update when the object saves successfully or not.

    describe ArticlesController do
      let(:article) { mock_model(Article) }

      describe 'POST create' do
        before { Article.stub(:new).and_return(article) }

        it 'creates a new article with the given attributes' do
          Article.should_receive(:new).with(title: 'The New Article Title').and_return(article)
          post :create, article: { title: 'The New Article Title' }
        end

        it 'saves the article' do
          article.should_receive(:save)
          post :create
        end

        context 'when the article saves successfully' do
          before { article.stub(:save).and_return(true) }

          it 'sets a flash[:notice] message' do
            post :create
            flash[:notice].should eq('The article was saved successfully.')
          end

          it 'redirects to the Articles index' do
            post :create
            response.should redirect_to(action: 'index')
          end
        end

        context 'when the article fails to save' do
          before { article.stub(:save).and_return(false) }

          it 'assigns @article' do
            post :create
            assigns[:article].should be_eql(article)
          end

          it 're-renders the "new" template' do
            post :create
            response.should render_template('new')
          end
        end
      end
    end
    ```

### Models

* The directory structure of the model specs in `spec/unit/models` matches the
  one in `app/models`. For example the specs for the model
  `app/models/article.rb` are placed in `spec/unit/models/article_spec.rb`.
* Do not mock the models in their own specs.
* Use factories to make real objects.
* It is acceptable to mock other models or child objects.
* Create the model for all examples in the spec to avoid duplication.

    ```Ruby
    describe Article do
      let(:article) { create(:article) }
    end
    ```

* Add an example ensuring that the generated model is valid.

    ```Ruby
    describe Article do
      it 'is valid with valid attributes' do
        article.should be_valid
      end
    end
    ```

* When testing validations, use `have(x).errors_on` to specify the attibute
which should be validated. Using `be_valid` does not guarantee that the problem
 is in the intended attribute.

    ```Ruby
    # bad
    describe '#title' do
      it 'is required' do
        article.title = nil
        article.should_not be_valid
      end
    end

    # prefered
    describe '#title' do
      it 'is required' do
        article.title = nil
        article.should have(1).error_on(:title)
      end
    end
    ```

* Add a separate `describe` for each attribute which has validations.

    ```Ruby
    describe Article do
      describe '#title' do
        it 'is required' do
          article.title = nil
          article.should have(1).error_on(:title)
        end
      end
    end
    ```

* When testing uniqueness of a model attribute, name the other object `another_object`.

    ```Ruby
    describe Article do
      describe '#title' do
        it 'is unique' do
          another_article = build(:article, title: article.title)
          article.should have(1).error_on(:title)
        end
      end
    end
    ```

### Mailers

* The model in the mailer spec should be mocked. The mailer should not depend on the model creation.
* The mailer spec should verify that:
  * the subject is correct
  * the sender e-mail is correct
  * the receiver(s) e-mail is stated correctly
  * the e-mail contains the required information

    ```Ruby
    describe SubscriberMailer do
      let(:subscriber) { mock_model(Subscription, email: 'johndoe@test.com', name: 'John Doe') }

      describe 'successful registration email' do
        let(:email) { SubscriptionMailer.successful_registration_email(subscriber) }

        it "has the correct value for the 'subject' attribute" do
          email.subject.should == 'Successful Registration!'
        end
        
        it "has the correct value for the 'from' attribute" do
          email.from.should == ['info@your_site.com']
        end
        
        it "has the correct value for the 'to' attribute" do
          email.to.should == [subscriber.email]
        end

        it 'contains the subscriber name' do
          email.body.encoded.should match(subscriber.name)
        end
      end
    end
    ```

### Uploaders

* What we can test about an uploader is whether the images are resized correctly.
Here is a sample spec of a [carrierwave](https://github.com/jnicklas/carrierwave) image uploader:

    ```Ruby

    # rspec/uploaders/person_avatar_uploader_spec.rb
    require 'spec_helper'
    require 'carrierwave/test/matchers'

    describe PersonAvatarUploader do
      include CarrierWave::Test::Matchers

      # Enable images processing before executing the examples
      before(:all) do
        UserAvatarUploader.enable_processing = true
      end

      # Create a new uploader. The model is mocked as the uploading and resizing images does not depend on the model creation.
      before(:each) do
        @uploader = PersonAvatarUploader.new(mock_model(Person).as_null_object)
        @uploader.store!(File.open(path_to_file))
      end

      # Disable images processing after executing the examples
      after(:all) do
        UserAvatarUploader.enable_processing = false
      end

      # Testing whether image is no larger than given dimensions
      context 'the default version' do
        it 'scales down an image to be no larger than 256 by 256 pixels' do
          @uploader.should be_no_larger_than(256, 256)
        end
      end

      # Testing whether image has the exact dimensions
      context 'the thumb version' do
        it 'scales down an image to be exactly 64 by 64 pixels' do
          @uploader.thumb.should have_dimensions(64, 64)
        end
      end
    end

    ```

## Unit, Integration, and Functional Tests

### Unit Tests

A unit test must focus on the output or the effect of one method called in isolation.
A unit test's expectation will test one of the following:
 * one characteristic of the return value
   * such as its data type, size, scalar value, ...
 * whether a particular method on a particular object was called
   * and optionally how many times it was called
   * and optionally what parameters it was called with
   * but never what that method's return value was
     * eg. never `stream_assembly.should_receive(:video_description).and_return(a_specific_video_description)`

### Integration Tests

An integration test ensures that parts of the application interact correctly.
It ensures that object interfaces are compatible.

Consider the following scenario:

 * a unit test exists that ensures that a node's service layer calls an API endpoint `/foo`, which is mocked

client code:

    def service(client)
      ... node node node ...
      client.call('/foo')
    end

client test:

    it 'calls the foo endpoint' do
        client = double(:client)
        client.should_receive(:call).with('/foo')
        service(client)
    end
 
 * another unit test exists that ensures that API endpoint `/foo` returns `bar`

api code:

    def foo
      'bar'
    end

api test:

    context 'any request' do
      it "returns 'bar'" do
        expect(api.foo).to eql('bar')
      end
    end

Now, we update the `/foo` API endpoint to require parameter `quux`.

api code:

    def foo(params)
      raise unless params[:quux]
      'bar'
    end

The API endpoint's unit test breaks, because `quux` is now required. So we update the unit test, and it passes.

api test:

    context 'a request with all required parameters' do
      it "returns 'bar'" do
        expect(api.foo(:quux => true) ).to eql('bar')
      end
    end

However, the node's service layer is still calling `/foo` without the required parameter,
_and the service layer's unit test is still passing_, because we mocked the endpoint.

An integration test would involve the complete API call's round trip, which excercises both sides of the API call in one transaction.

An integration test that starts in the node's service layer and calls the API endpoint without the required parameter would fail.

integration test:

    context 'a client calling the api' do
      it 'returns bar' do
        expect(client.call('/foo') ).to eql 'bar'
      end
    end

...and would remind us to update the client test

client test:

    it 'calls the foo endpoint with the correct parameters' do
        client = double(:client)
        client.should_receive(:call).with('/foo?quux=true')
        service(client)
    end

...and update the client code

client code:

    def service(client)
      ... node node node ...
      ... maybe even set the quux value correctly ...
      client.call("/foo?quux=#{quux}")
    end

...and finally update the integration test:

    context 'a client calling the api with the correct parameters' do
      it 'returns bar' do
        expect(client.call('/foo?quux=true') ).to eql 'bar'
      end
    end

### Functional Tests

A functional test ensures that the application is meeting the business needs of the customer.
It ensures that user stories are being carried out consistently and completely.

Functional tests often take the form of integration tests, but by definition they test the product's features that the customer specifically relies on.

This is generally where something like Cucumber comes in...

# Further Reading

There are a few excellent resources on Rails style, that you should
consider if you have time to spare:

* [The Rails 3 Way](http://www.amazon.com/Rails-Way-Addison-Wesley-Professional-Ruby/dp/0321601661)
* [Ruby on Rails Guides](http://guides.rubyonrails.org/)
* [The RSpec Book](http://pragprog.com/book/achbd/the-rspec-book)
* [The Cucumber Book](http://pragprog.com/book/hwcuc/the-cucumber-book)
* [Everyday Rails Testing with RSpec](https://leanpub.com/everydayrailsrspec)

# License

![Creative Commons License](http://i.creativecommons.org/l/by/3.0/88x31.png)
This work is licensed under a [Creative Commons Attribution 3.0 Unported License](http://creativecommons.org/licenses/by/3.0/deed.en_US)
[Bozhidar](https://twitter.com/bbatsov)
