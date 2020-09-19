# ActAsInteractor
Simple and powerful interface for creating service objects in Ruby.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'act_as_interactor'
```

And then execute:

    $ bundle install

Or install it yourself as:

    $ gem install act_as_interactor

## Usage
Just include the interactor module in your service objects. For example if you have a service objects for creating blog posts in your app, then just include the `AtcAsInteractor` module in it and define the `execute` method inside it:

```ruby
module Posts
  class Create
    include ActAsInteractor

    def execute(params)
      # steps to create a blog post
    end
  end
end
```

NOTE: the params argument of the execute is better to be a hash of parameters.

## defining steps
For defining steps needed for your operation, just add simple ruby methods to your service objects and pass the params hash to it. Then, using destructuring the params hash, get the values needed for that specific step and ignore the rest.

```ruby
module Posts
  class Create
    include ActAsInteractor

    def execute(params)
      yield create_post(params)
    end

    private

    def create_post(title:, body:, **)
      post = Post.create(title: title, body: body)
      
      return Failure(post.errors.messages) if post.invalid?

      Success(post)
    end
  end
end
```
NOTE: We use `Failure` and `Success` methods to wrap our steps results. Then we can use `yield` keyword to unwrap the output of the method. It also helps to halt the execution of the service objects if there is a `Failure` in execution of any of steps and returns the Failure object wrapping the real output.

## Input validation
For validating the input, you only need to add a `validator` method to your service object that returns an object that responds to call method and receives a hash of parameters. If you define the validator method, then the validation process starts automatically before executing any step in the srevice object.

Adding validation using `dry-schema`:
```Ruby
module Posts
  class Create
    include ActAsInteractor

    def execute(params)
      yield create_post(params)
    end

    private

    def create_post(title:, body:, **)
      post = Post.create(title: title, body: body)
      
      return Failure(post.errors.messages) if post.invalid?

      Success(post)
    end

    def validator
      Dry::Schema.Params do
        required(:title).filled(:str?)
        required(:body).filled(:str?)
      end
    end
  end
end
```

You can also use more sophisticated validation tools like `dry-validation`:
```Ruby
module Posts
  module Contract
    class Create < Dry::Validation::Contract
      params do
        required(:title).filled(:str?)
        required(:body).filled(:str?)
        required(:author_email).filled(:str?)
      end

      rule(:author_email) do
        unless /\A[\w+\-.]+@[a-z\d\-]+(\.[a-z\d\-]+)*\.[a-z]+\z/i.match?(value)
          key.failure('has invalid format')
        end
      end
    end
  end
end


module Posts
  class Create
    include ActAsInteractor

    def execute(params)
      yield create_post(params)
    end

    private

    def create_post(title:, body:, **)
      post = Post.create(title: title, body: body)
      
      return Failure(post.errors.messages) if post.invalid?

      Success(post)
    end

    def validator
      Posts::Contract::Create.new
    end
  end
end
```
## Getting the result
As we've wrapped all the steps results in `Failure` or `Success` objects, then we have access to `failure?` and `success?` methods to see if the output of the serivce is successful or a failure. We also have `failure` and `success` methods to get the unwrapped output of the service objects:

```ruby
outcome = Posts::Create.call(title: "Hello world") # => Success(#<Post ...>)
outcome.success? # => true
outcome.success # => #<Post ...>
```
But wait! There's also a more interesting way to get the service object outcome, just pass a block. In this case, I want to use it inside a Rails controller action for exapmle:
```ruby
class PostsController
  def create
    Posts::Create.call(post_params.to_h) do |outcome|
      outcome.success do |post|
        render json: post, status: :ok
      end

      outcome.failure do |errors|
        render json: errors, status: :unprocessable_entity
      end
    end
  end
end
```

## Better error handling (Railway Oriented)
Based on the monadic type of the result of your service objects using this library, you can easily handle your different failure paths in a pretty shape without using if-else statements or throwing different types of exceptions (read more about Railway Oriented Programming).

```ruby
module Posts
  class Create
    include ActAsInteractor

    def execute(params)
      # ...
      yield check_title(params)
      # ...
    end

    private

    def create_title(title:, **)
      outcome = Posts::CheckTitle.call(title: title)

      return Failure(:inappropriate_title) if outcome.failure?

      return Success()
    end

    # ...
  end
end
```
Then in the caller method, you can check different failure paths:

```ruby
class PostsController
  def create
    Posts::Create.call(post_params.to_h) do |outcome|
      # success path
      outcome.success do |post|
        render json: post, status: :ok
      end

      # inappropriate title failure path
      outcome.failure(:inappropriate_title) do |_errors|
        # do something like send a mail or notify admins etc.
        render json: { errors: { title: ["is inappropriate"] }}
      end

      # general failure path
      outcome.failure do |errors|
        render json: { errors: errors }, status: :unprocessable_entity
      end
    end
  end
end
```
## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/arefaslani/act_as_interactor. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [code of conduct](https://github.com/arefaslani/act_as_interactor/blob/master/CODE_OF_CONDUCT.md).


## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).

## Code of Conduct

Everyone interacting in the ActAsInteractor project's codebases, issue trackers, chat rooms and mailing lists is expected to follow the [code of conduct](https://github.com/arefaslani/act_as_interactor/blob/master/CODE_OF_CONDUCT.md).
