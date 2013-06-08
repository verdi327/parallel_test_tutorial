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
Capybara is a great tool that gives a nice DSL for interacting with DOM elements i.e finding, selecting, clicking, filling in form...you know, all things your users are going to be doing.  Luckily for us, the `cucumber-rails` gem that we installed already has `capybara` built in.

Inside of the `features` directory make a new directory called `tests`.  Inside of that directory create a new `.feature` file called `smoketest.feature`.



