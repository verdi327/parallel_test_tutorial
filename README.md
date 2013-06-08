#Parallel Cross Browser Testing       
###Using Saucelabs, Cucumber, and Capybara

##Goal
In this tutorial we will create a rails app from scratch that contain `cucumber .feature` files that we can run in parallel against 6 different browsers.  Cross browser testing is super important because each browser has its own implementation on how it handles rendering the `DOM` and as developers we want to ensure that our product works with all the browsers our customers are using.  

The type of tests we will be covering today are called integration tests.  These tests are fully headed, meaning they simulate the actions of real users by automating the actions of the browser.  Since many sites today use `javascript` we'll be using the `Selenium Webdriver` to handle these interactions.  Because many large scale apps today follow a `Service Oriented Architecture` we'll be building a rails app the solely houses `cucumber .feature` files and their corresponding `step definitions` which we can then run remotely against other applications.

To implement cross browser testing we'll need to spin up multiple virtual machines where each is configured with the correct OS and browser.  Such a task is outside the range of this tutorial.  Instead, we'll be leveraging the awesome tools provided by SauceLabs.  They have already handled the hard part of provisioning VMs so all we need to do is write our tests and send them to sauce.  Moving forward in this tutorial, you'll need a SauceLabs account, which you can sign up for free [here](http://saucelabs.com/ "saucelabs").

##Phase 1: Getting Cucumber Up and Running
Create a new rails app called super_test
```
rails new super_test
```

Install the necessary gems 
```
gem rails-cucumber, :require => false
```
Bundle

Now run the Cucumber generator
```
rails generate cucumber:install
```

It should have generated a `features` folder and created the `cucumber command`

To make sure we're not crazy let's run `cucumber`. You should get the follow output:
```
Using the default profile...
0 scenarios
0 steps
0m0.000s
```
Great we now have `Cucumber` wired up correctly.

##Phase 2: Adding Capyabra
Capybara is a great tool that gives a nice DSL for interacting with DOM elements i.e finding, selecting, clicking, filling in form...you know, all things your users are going to be doing.  Luckily for us, the `cucumber-rails` gem that we installed already has `capybara` built in.  First, let's get some configuration at of the way.

Open the `env.rb` file inside of `support` directory.  Add the following to the top of the file.

```
require 'cucumber/rails'

Capybara.run_server = false
Capybara.register_driver(:selenium){ |app| Capybara::Selenium::Driver.new(app, { :browser => :chrome }) }
Capybara.default_driver = :selenium

```
The `Capybara.run_server = false` method tells Capybara that we are running aginst a remote server.
The next 2 lines tell Capybara that we want to use the `Selenium Webdriver` and we want it to run `Chrome` as opposed to the default `firefox` (just my own personal preference)

Now, let's create a more robust smoketest.  Inside of the `features` directory make a new directory called `tests`.  Inside of that directory create a new `.feature` file called `smoketest.feature`.

Add this to it:
```
Feature: Testing different webpages
  Scenario: Around the world
    Given I am checking out many pages
```

Now run `cucumber` from the command line and it should generate the following:

```
Using the default profile...
Feature: Testing different webpages

  Scenario: Around the world           # features/test/external.feature:3
    Given I am checking out many pages # features/test/external.feature:4
      Undefined step: "I am checking out many pages" (Cucumber::Undefined)
      features/test/external.feature:4:in `Given I am checking out many pages'

1 scenario (1 undefined)
1 step (1 undefined)
0m0.358s

You can implement step definitions for undefined steps with these snippets:

Given(/^I am checking out many pages$/) do
  pending # express the regexp above with the code you wish you had
end
```

Great, Cucumber is giving us the necessary `step definition` to complete this test.  So, let's create a new `.rb` file inside of the `step_definitions` directory called `smoketest.rb`.  Inside of that file, add the following:

```
Given(/^I am checking out many pages$/) do
  g = "https://www.google.com"
  l = "https://www.livingsocial.com"
  t = "https://www.tumblr.com"
  [g,l,t].each do |site|
    visit site
    sleep 2
  end
end
```

Capybara gives us the method `visit` and all we do is pass it url.  Now, we should be able to run `cucumber` and see the following:

```
Using the default profile...
Feature: Testing different webpages

  Scenario: Around the world           # features/test/external.feature:3
    Given I am checking out many pages # features/step_definitions/smoketest.rb:1

1 scenario (1 passed)
1 step (1 passed)
0m12.931s
```

Great we now have capybara wired up.







