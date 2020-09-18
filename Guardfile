clearing :on

guard :shell do
  watch(/(act_as_interactor.gemspec|lib\/.*\.rb)/) do |m|
    output = []
    output << `bundle install`
    output << `gem build act_as_interactor.gemspec`
    output << `gem install act_as_interactor-#{ActAsInteractor::VERSION}.gem`
    output.join("\n")
  end
end
