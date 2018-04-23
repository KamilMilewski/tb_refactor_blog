## Trailblazer Operation - refactor existing Rails codebase - step by step guide. PART I

So you heard about [Trailblazer](http://trailblazer.to/), how it can help organize your code(new and existing one). You read some official guide, and some other articles here and there. You heard how Trailblazer allows you to gently refactor existing legacy app step by step. Today we are going to put this claim to the test. To be exact - we will rewrite class written with service object pattern to be a [Trailblazer operation](http://trailblazer.to/guides/trailblazer/2.0/01-operation-basics.html).

#### Brief introduction to the app at hand
In our app there are challenges to which users can join. Participation is a model holding info about user... participation in given challenge. Business logic behind this process is encapsulated in service object invoked in [Grape](https://github.com/ruby-grape/grape#what-is-grape) endpoint. Though we don't have unit tests for service object, we have integration tests for participations endpoint on which we will rely for now (later we will write unit tests for operation and possibly skinny down endpoint integration tests as well). Here is how it all looks like at the moment:

Participations create service object:
```ruby
  module Api
    module Challenges
      module Participations
        class Create

          JoiningBlockedError = Class.new(StandardError)
          DuplicatedParticipationError = Class.new(StandardError)

          def self.call(params)
            new(params).call
          end

          def initialize(params)
            @params = params
          end

          def call
            raise JoiningBlockedError unless can_join?
            raise DuplicatedParticipationError if user_duplicated_participation?
            participation = nil
            ActiveRecord::Base.transaction do
              participation = Participation.create!(participation_params)
              if challenge.open || challenge.sponsored
                participation = Api::Challenges::Accept.call(challenge_id: challenge.id, user_id: participation.user_id)
              end
            end
            Challenges::UpdateStatus.new(participation.challenge.id).call
            if participation.pending? && participation.user_id != challenge.creator_id
              Notifications::Challenges::ParticipationPending.call(participation: participation)
            end
            participation
          end

          private

          attr_reader :params

          def user_duplicated_participation?
            Participation.where(
              user_id: params[:user_id], challenge_id: challenge.id
            ).exists?
          end

          def can_join?
            return false if challenge.participations_count >= 2 && !challenge.sponsored
            challenge.submission_ends_at.nil? || DateTime.now < challenge.submission_ends_at
          end

          def challenge
            @challenge ||=
              if params[:invitation_token]
                Challenge.find_by!(invitation_token: params[:invitation_token])
              else
                Challenge.find(params[:challenge_id])
              end
          end

          def participation_params
            params.slice(:user_id, :acceptation_status).merge(challenge_id: challenge.id)
          end
        end
      end
    end
  end
```

Details of business logic contained here are really not that important but we have a few points worth noting:
- we have flow control based on exceptions:
```ruby
raise JoiningBlockedError unless can_join?
raise DuplicatedParticipationError if user_duplicated_participation?
```
- we also run other service objects in this one:
  ```ruby
  Challenges::UpdateStatu
  ```
  ```ruby
  Notifications::Challenges::ParticipationPending
  ```
  and
  ```ruby
  Api::Challenges::Accept
  ```
- as a result this service object returns participation record.

Below is Grape endpoint which runs code above. If You are not familiar with Grape don't worry, it doesn't really make a difference in our case - it could be as well regular rails controller.
```ruby
  post do
    participation = Api::Challenges::Participations::Create.call(
      declared(params).merge(user_id: current_user.id)
    )
    present participation, with: V1::Entities::Participation

  rescue Api::Challenges::Participations::Create::JoiningBlockedError
    error!({ error_code: 100, error_message: 'Joining to challenge is disabled' }, 400)
  rescue Api::Challenges::Participations::Create::DuplicatedParticipationError
    error!({ error_code: 101, error_message: 'Participation already exist' }, 400)
  end
```

#### Let's dive right into the code

First we have to add some gems:
```ruby
gem "trailblazer", "2.0.3"
gem "trailblazer-rails"
```
Notice we are specifying TB version 2.0.3. This is quite important as TB is under active development and a lot has changed since 1.0. There are also few important differences in 2.1.

##### Iteration I: operation - let it just work

Following TB convention we organize our file structure by domain(unlike in Rails where it is by technology - for example, models and controllers directories). So for the beginning, let's create our li'l cosy spot for our new operation:
```
app/concepts/challenges/participations/operations/create.rb
```
It is TB convention to put all its code into ```app/concepts``` directory. Then we divide it by domain concepts ```challenges/participations```. Finally we create directories for related TB concepts - in our case just ```operations```.


Now lets define empty trailblazer operation class with one almighty step:
```ruby
class Challenges::Participations::Create < Trailblazer::Operation
  step :do_everything!

  def do_everything!(options, params:)

  end
end
```
Now we replace service object in endpoint with our newly created operation:

```ruby
  post do
    result = ::Challenges::Participations::Create.(
      declared(params).merge(user_id: current_user.id)
    )
    participation = result['model']
    present participation, with: V1::Entities::Participation
  rescue ::Challenges::Participations::Create::JoiningBlockedError
    error!({ error_code: 100, error_message: 'Joining to challenge is disabled' }, 400)
  rescue ::Challenges::Participations::Create::DuplicatedParticipationError
    error!({ error_code: 101, error_message: 'Participation already exist' }, 400)
  end
```
It is almost identical to previous form. The main difference lies in service object and operation return value.
For service object return value was just created participation:
```ruby
participation = Api::Challenges::Participations::Create.call(
  declared(params).merge(user_id: current_user.id)
)
```
For TB operation on the other hand it will be so called result object:
```ruby
result = ::Challenges::Participations::Create.(
  declared(params).merge(user_id: current_user.id)
)
participation = result['model']
```
In this result object, we intend to store participation record under ```'model'``` key.
Of course, specs for participations endpoint are all red now as we yet do not have any logic in it. So, back to the operation - let's copy a few things from service object:
- ```process``` method body to ```do_everything!``` step in operation
- exception constants ```JoiningBlockedError``` and ```DuplicatedParticipationError```
- all private methods to operation class

Now our operation looks like this:

```ruby
class Challenges::Participations::Create < Trailblazer::Operation
  JoiningBlockedError = Class.new(StandardError)
  DuplicatedParticipationError = Class.new(StandardError)

  step :do_everything!

  def do_everything!(options, params:)
    raise JoiningBlockedError unless can_join?
    raise DuplicatedParticipationError if user_duplicated_participation?
    participation = nil
    ActiveRecord::Base.transaction do
      participation = Participation.create!(participation_params)
      if challenge.open || challenge.sponsored
        participation = Api::Challenges::Accept.call(challenge_id: challenge.id, user_id: participation.user_id)
      end
    end
    Challenges::UpdateStatus.new(participation.challenge.id).call
    if participation.pending? && participation.user_id != challenge.creator_id
      Notifications::Challenges::ParticipationPending.call(participation: participation)
    end
    participation
  end

  def user_duplicated_participation?
    Participation.where(
      user_id: params[:user_id], challenge_id: challenge.id
    ).exists?
  end

  def can_join?
    return false if challenge.participations_count >= 2 && !challenge.sponsored
    challenge.submission_ends_at.nil? || DateTime.now < challenge.submission_ends_at
  end

  def challenge
    @challenge ||=
      if params[:invitation_token]
        Challenge.find_by!(invitation_token: params[:invitation_token])
      else
        Challenge.find(params[:challenge_id])
      end
  end

  def participation_params
    params.slice(:user_id, :acceptation_status).merge(challenge_id: challenge.id)
  end
end
```

Looks ugly and promising. Unfortunately there are a few problems
1. Our service object has constructor ```initialize``` and attribute reader which our operation lacks. Thus all private methods won't have access to ```params``` instance variable. To solve this we resort to just explicitly passing params to every method. Like this:
```ruby
raise JoiningBlockedError unless can_join?(params)
```
```ruby
def can_join?(params)
```
It's not pretty but it works and that is what we aim for now ;)

2. Service object has return value which is participation at the end of the process method. In case of TB operation step, this variable will just evaluate to true at the and of the step method to indicate step success or failure.
In TB each step can write to the options object and in the end this object will be available in the result of the operation invocation. This is the TB way to communicate operation internal state between steps, which we do by passing it as the first argument to each step method we define:
```ruby
def do_everything!(options, params:)
```
In step we can write to options object, in our case it will be as follows:
```ruby
options['model'] = participation
```

3. Everything looks fine and dandy but when we run related specs we have little nasty surprise:
```ruby
NameError:
  uninitialized constant Challenges::Comments
```
Looks like we have to load dependencies used in our TB operation manually. To do so:
```ruby
require_dependency Rails.root.join 'app', 'api', 'OurLegacyApp', 'v1.rb'
```

```require_dependency``` is the Rails way to load dependencies and having them reloaded as needed.

In the end our operation looks like this:
```ruby
require_dependency Rails.root.join 'app', 'api', 'OurLegacyApp', 'v1.rb'

class Challenges::Participations::Create < Trailblazer::Operation
  JoiningBlockedError = Class.new(StandardError)
  DuplicatedParticipationError = Class.new(StandardError)

  step :do_everything!

  def do_everything!(options, params:)
    raise JoiningBlockedError unless can_join?(params)
    raise DuplicatedParticipationError if user_duplicated_participation?(params)
    participation = nil
    ActiveRecord::Base.transaction do
      participation = Participation.create!(participation_params(params))
      if challenge(params).open || challenge(params).sponsored
        participation = Api::Challenges::Accept.call(challenge_id: challenge(params).id, user_id: participation.user_id)
      end
    end
    Challenges::UpdateStatus.new(participation.challenge.id).call
    if participation.pending? && participation.user_id != challenge(params).creator_id
      Notifications::Challenges::ParticipationPending.call(participation: participation)
    end
    options['model'] = participation
    true
  end

  def user_duplicated_participation?(params)
    Participation.where(
      user_id: params[:user_id], challenge_id: challenge(params).id
    ).exists?
  end

  def can_join?(params)
    return false if challenge(params).participations_count >= 2 && !challenge(params).sponsored
    challenge(params).submission_ends_at.nil? || DateTime.now < challenge(params).submission_ends_at
  end

  def challenge(params)
    @challenge ||=
      if params[:invitation_token]
        Challenge.find_by!(invitation_token: params[:invitation_token])
      else
        Challenge.find(params[:challenge_id])
      end
  end

  def participation_params(params)
    params.slice(:user_id, :acceptation_status).merge(challenge_id: challenge(params).id)
  end
end
```

Specs are green, thus first iteration of our refactor is done. For now there is not much a difference between good old service object and operation. You could even say its worse than before and You would be probably right. But we are far from over :) In the ongoing post we will slim down our __FATTY__ ```do_everything!``` step by dividing it to more smaller steps.
In the process we will also get rid of exception based flow control. [Go to Part II](#)
