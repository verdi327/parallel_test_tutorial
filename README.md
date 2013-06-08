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

##Phase 3: Running on SauceLabs

We now want to be able to run our test on SauceLabs using their OnDemand platform.  First, let's add some more gems.
```
gem "sauce-cucumber", :require => false
```
Bundle - And yes the `:require => false` is necessary!

Now let's get some config out of the way. Inside of the `env.rb` file add the following:
```
require "sauce/cucumber"
Capybara.javascript_driver = :sauce

Sauce.config do |c|
  c[:browsers] = [['Mac', 'Chrome', '']]
end
```

So, as of now, your `env.rb` file should look like the following:
```
require 'cucumber/rails'
require "sauce/cucumber"


Capybara.run_server = false
Capybara.register_driver(:selenium){ |app| Capybara::Selenium::Driver.new(app, { :browser => :chrome }) }
Capybara.default_driver = :selenium
Capybara.javascript_driver = :sauce

Sauce.config do |c|
  c[:browsers] = [['Mac', 'Chrome', '']]
end
```
**Let's break this down**

* Because of file loading issues, you need to require `sauce/cucumber` after `cucumber/rails`.
* `Capybara.javascript_driver = :sauce` tells Capybara that any tests that need to be run with `javascript` should be handled by sauce.  What sauce has done is any test tagged with `@selenium` will be run automatically on the Sauce OnDemand platform.
* The `Sauce.config` block tells sauce what platform(Mac), what browser(Chrome), and what browser version(chrome doesn't have versions so you just put an empty string.  And yes, it has to be in an array of arrays!

We now need to add our creditentials so SauceLabs knows who we are.  Inside of our `config` directory create a new `.yml` file called `ondemand.yml`.  The sauce gem is looking for this file.  Add the following:

```
access_key: YOUR_ACCESS_KEY_HERE  
username: YOUR_USERNAME_HERE
```

Now, we need to go to the top of our `smoketest.feature` file and add the `@selenium` tag so we can indicate what tests we want run on Sauce OnDemand.  You file should look like this:

```
@selenium
Feature: Testing different webpages

  Scenario: Around the world
    Given I am checking out many pages
```

Now, run `cucumber` from you command line and you should be running the test on SauceLabs.

##Phase 4: Parallelize the Test (part 1)

Time for the fun part.  Before we code it, let's take a step back and outline the design we will be using to achieve parallelization.  Every time we run `cucumber` it loads the `env.rb` file and then runs all the tests we tell it to run.  Our `env.rb` file contains the os, browser, and brower version we want sauce to run.  So, we'll create our own rake task that will loop thru all of the os/browser/version configs we want, set that inside of the `env.rb` file and then run the process in background.  Essentailly, each config we have represents its own thread.  To accomplish this, we'll use `Thor` to dynamically change our `env.rb` file for us.

Add the necessary gems
```
gem thor
```
Bundle

**One thought**

One approach using `thor` would be to create multiple `env.rb` files where each contains the setup that we want.  Inside of our rake task we would just copy the file over and set it as our `env.rb` file.  However, this feels too bloated for me.  As a project grows you can have anywhere from 10-20 different configs which only have a few lines changed in each file - non very DRY.  In addition, you have to do some hackery by appending non `.rb` file extensions so that `cucumber` doesn't load them at runtime (if you place them within the `features` directory).

**A Better Solution**

A better approach is to just have one `env.rb` file where you only change what needs to be changes.  In our case it's this line:
```
Sauce.config do |c|
  c[:browsers] = [['Mac', 'Chrome', '']] #=> only this line needs to change
end
```

`Thor` has a method called `gsub_file` which allow us to find and replace within a file.  The method signature is below:
```
gsub_file(path, flag, *args, &block)

path:  path of the file to be changed
flag:	the regexp or string to be replaced
replacement:	the replacement, can be also given as a block
config:	give :verbose => false to not log the status.
```

So, inside of our `lib/tasks` directory let's create a new file called `set.thor`.  Add the following:
```
class Set < Thor
  include Thor::Actions

  desc "browser", "sets the proper browser/platform config into cucumber's env.rb for sauce labs"
  method_option :values, :type => :hash
  def browser
    gsub_file("features/support/env.rb", /c\[:browsers\] = \[\[[^\]]*\]\]/,"c[:browsers] = [['#{options[:values]["platform"]}', '#{options[:values]["browser"]}', '#{options[:values]["version"]}']]")
  end
end
```

**Breakdown**
* `include Thor::Actions` is the Thor module that gives us access to the gsub_file method
* `desc` same as `rake` just describes what the action does
* `method_option` denotes how we are going to pass arguments from the command line into our method
* Then we simply pass it the file, what to look for, and then what to add.

Now, we should be able to run the following from the command line:
```
thor set:browser --values=platform:"Mac" browser:"Firefox" version:"21"

```

And see this output `gsub  features/support/env.rb`

Now if we look at our `env.rb` file, the `Sauce.config` block should have changed to the following:
```
Sauce.config do |c|
  c[:browsers] = [['Mac', 'Firefox', '21']]
end
```

Awesome, we now have a way to dynamically change our `env.rb` file.

##Phase 5: Parallelize the Test (part 2)

We now need to create a file that contains all our browser configs.  Inside of the `support` directory create a new `.yml` file called `sauce_browsers.yml`.  Add the following to it:

```
list:
  - ["Windows XP","Internet Explorer","7"]
  - ["Windows 7","Internet Explorer","8"]
  - ["Windows 7","Internet Explorer","9"]
  - ["Mac", "Firefox", "21"]
  - ["Mac", "Safari", "5"]
  - ["Mac", "Chrome", ""]
```

*These are the configs I am interested in running, yours maybe different*

Now we need to create a new rake tests that will kick this party off!  Inside of `lib/taks` directory create a new `.rake` task called `sauce.rake`.  Add the following:

```
namespace :sauce do
  desc "Runs cucumber suite on Sauce OnDemand cross-browser"
  task :cucumber do
    browsers = YAML.load_file(File.join("config", "sauce_browsers.yml"))
    browsers['list'].each do |browser|  
      system("thor set:browser --values=platform:'#{browser[0]}' browser:'#{browser[1]}' version:'#{browser[2]}'")
      pid = fork { exec("cucumber") }
      Process.detach(pid)
      sleep 2
    end
  end
end
```

**Breakdown**
* First we namespace the task so that we can ultimately run `rake sauce:cucumber`
* Create an array of browser configs by loading the `sauce_browsers.yml` file
* Loop thru each browser, call our `thor set:browser` task and pass it the necessary args.
* Call `fork { exec("cucumber") }` which spins up a new thread and then `Process.detach` which makes the thread asynch so we don't have to wait for it to complete before yielding control back to our current process.


Great, now when we run `rake sauce:cucumber` our tests will run in 6 differnt browsers on SauceLabs.

##Phase 6: Running Against Localhost

We have parallelized our test, but our test is pretty lame.  It's an external process that just visits a few webpages.  When we're developing, we want to be able to test against our local environment.  Our current setup won't allow for that.  Luckily SauceLabs has again already done the hard work for us.  The have created `SauceConnect` which creates a tunnel between our local test environment and the VMs that are running our tests.

To get going, install the necessary gems

```
gem sauce-connect
```
Bundle

Add some more config to our `env.rb` file
```
Capybara.server_port = 80

And this inside our Sauce.config block
  c[:start_tunnel] = false
```

Our final `env.rb` file should look like this:
```
require 'cucumber/rails'
require "sauce/cucumber"


Capybara.run_server = false
Capybara.register_driver(:selenium){ |app| Capybara::Selenium::Driver.new(app, { :browser => :chrome }) }
Capybara.default_driver = :selenium
Capybara.javascript_driver = :sauce
Capybara.server_port = 80

Sauce.config do |c|
  c[:browsers] = [['Mac', 'Chrome', '']]
  c[:start_tunnel] = true
end
```

**Breakdown**
* `Capybara.server_port` is needed because Capybara runs its server on a random port, whereas SauceConnect is expecting a port from a specific range.  Port 80 is from that range.
* `c[:start_tunnel] = true` Tells sauce that it should spin up the SauceConnect tunnel before the test is run.

Time to write a test.  At this point I can give you a sample because each of our dev env's will be different.  But, you should create a new `.feature` file within the `features/test` directory.  A quick smoketest could to hit your landing page and click on some button, like `login`.  Remember to tag the test with `@selenium` so that sauce will pick it up.

Awesome Sauce - should be good running against your dev env on sauce labs in 6 different browsers in parallel.
Enjoy!

##Additonal Notes

**Run Sauce by Default**

If you want to just run all of your tests against SauceLabs all the time change your `env.rb` file to look like this:
```
require 'cucumber/rails'
require "sauce/cucumber"

Capybara.run_server = false
Capybara.default_driver = :sauce
Capybara.server_port = 80

Sauce.config do |c|
  c[:browsers] = [['Mac', 'Chrome', '']]
  c[:start_tunnel] = true
end
```

**Long Running Sauce-Connect Tunnel**

If you're using SauceConnect heavily, you'll want to make it so that you always have the tunnel running.  In our current implementation, the tunnel will get spun up and torn down between each test run.  This is *very* taxing as the tunnel takes between 10-20 seconds to spin up and 5-10 to close.  A better solution is to keep the tunnel running constantly in another thread.

**Manually*

The easiest way to achieve that is to download the sauce-connect.jar file, add it into your app directory and then run it in another console tab.  SauceLabs has put together an easy tutorial on how to downlaod and run [here](https://saucelabs.com/docs/connect "sauce-connect")

**Progammatically**

Or, you could create a SauceConnect class that deals with downloading, starting, and stopping the tunnel which you could then use as part of bootstrap script so you can run your tests with one command.

Inside of your `support` directory create a new `.rb` file called `sauce_connect.rb`. Add the following:

```
class SauceConnect

  def self.configuration_file
    File.join(File.dirname(__FILE__),"..","..","config", "ondemand.yml")
  end

  def self.configuration
    YAML.load_file(configuration_file)
  end

  def self.username
    configuration['username']
  end

  def self.access_key
    configuration['access_key']
  end

  def self.archive_file
    "Sauce-Connect-latest.zip"
  end

  def self.application_file
    "Sauce-Connect.jar"
  end

  def self.connect_url
    "http://saucelabs.com/downloads/#{archive_file}"
  end

  def self.download
    `wget #{connect_url}`

    unless File.exists?(archive_file)
      raise error_message("could not download the Sauce Archive File",
        "Attempted to download it from:\n  #{connect_url}")
    end
  end  
  
  def self.is_open?
    !%x{ps | grep Sauce-Connect.jar | head -n 1}.include?("grep")
  end


  def self.extract
    `unzip -o #{archive_file} #{application_file}`

    unless File.exists?(application_file)
      raise error_message("could not extract the Sauce Application File",
        "Attempted to extract:\n    File: #{application_file}\n    From: #{archive_file}")
    end

  end

  def self.start
    if is_open?
      puts "Tunnel is already open!"
      exit
    end
    puts "Starting the Sauce Connect Tunnel"
    pid = fork { exec("java -jar #{application_file} #{username} #{access_key} &> #{log_file}") }
    Process.detach(pid)
    sleep 2
    connected = false
    until connected
      if File.read(log_file).include?("Connected! You may start your tests")  && is_open?
        puts ""
        puts "Connected! Tests are ready to run"
        connected = true
      else
        sleep 2
        print "."
      end
    end
  end

  def self.end
    unless is_open?
      puts "Tunnel is already closed!"
      exit
    end
    puts "Gracefully shutting down the Sauce Tunnel"
    exec("touch #{log_file}") unless File.exists? log_file #there are cases where a bad connect will not create the log file
    system "kill -TERM $(ps | grep Sauce-Connect.jar | head -n 1 | awk '{print $1}')"
    terminated = false
    until terminated
      if File.read(log_file).include?("Finished /")  && !is_open?
        puts "\nSuccess!"
        system("rm #{log_file}") 
        terminated = true
      else
        sleep 2
        system "kill -TERM $(ps | grep Sauce-Connect.jar | head -n 1 | awk '{print $1}')"
        print "."
      end
    end
  end      

  def self.log_file
    File.join(Dir.pwd, "log", "sauce_connect.log")
  end

  def self.cleanup
    `rm #{archive_file}`
  end

  def self.error_message(error,info)
    %{
====================================================
 ! ERROR - #{error}

  #{info}

   Please visit Sauce Labs!
====================================================}
  end

end
```

This is a lot of code.  But, what it gives us access to 5 class methods.
```
SauceConnect.download
SauceConnect.extract
SauceConnect.cleanup
SauceConnect.start
SauceConnect.end
```
* `Download, Extract, Cleanup` will download the latest stable version from SauceLabs and install in our app's directory, unzip it, and then delete the original zip file so we only have the .jar file.
* `Start` will start the tunnel in its own process so you don't need to open a second tab on the command line
* `End` will find the process's pid and kill it (SIGTERM)

Now we can wrap this functionality into their own `rake` tasks.  Add the following to your `sauce.rake` file.

```
  desc "Installs (overwriting) the necessary Sauce Connect Library"
  task :install do

    SauceConnect.download
    SauceConnect.extract
    SauceConnect.cleanup

    puts "Successfully downloaded and extracted the SauceConnect Library"
  end
  
  desc "Opens a Sauce Connect Tunnel"
  task :open_tunnel do
    SauceConnect.start
  end

  desc "Gracefully closes the Sauce Connect Tunnel"
  task :close_tunnel do
    SauceConnect.end
  end
```

Woot Woot! We now have 3 helpful `rake` tasks for dealing with SauceConnect Tunnel.


















