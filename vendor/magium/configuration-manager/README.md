[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/magium/configuration-manager/badges/quality-score.png?b=develop)](https://scrutinizer-ci.com/g/magium/configuration-manager/?branch=develop) 
[![Code Coverage](https://scrutinizer-ci.com/g/magium/configuration-manager/badges/coverage.png?b=develop)](https://scrutinizer-ci.com/g/magium/configuration-manager/?branch=develop)
[![Build Status](https://scrutinizer-ci.com/g/magium/configuration-manager/badges/build.png?b=develop)](https://scrutinizer-ci.com/g/magium/configuration-manager/build-status/develop)
# The Magium Configuration Manager (MCM)

The Magium Configuration Manager provides a configuration setup that is similar to Magento's configuration management.  There are 3 purposes for this library:

1. Provide a standardized configuration panel/CLI so developers don't need to do a code deployment to change a configuration value.
2. Provide an inheritable interface where values can "bubble up" through child contexts
3. Provide a unified (read merged) configuration interface for third parties that work in your application like magical unicorn dust.

In order for it to work for you it needs 3 things

1. A few different configuration files (more details later)
2. A database adapter to store persistent configuration values
3. A cache adapter to store the processed configuration objects

If you want to see it in action, instead of reading about it, check out the introductory YouTube video.

[![Magium Configuration Manager ](http://img.youtube.com/vi/76MLD9Kl2Lk/0.jpg)](http://www.youtube.com/watch?v=76MLD9Kl2Lk "Magium Configuration Manager ")

## Getting Started
### Wiring Configuration

The core object is the [`Config`](lib/Config/Config.php) object.  That is what your application will typically interact with.  The intent is that all you need to do is require a `Config` object as a dependency and it all works out nicely.  

In other words, this:

```
use Magium\Configuration\Config\Config;

class Pinky
{

    protected $brain;
    
    public function __construct(Config $brain) 
    {
        $this->brain = $brain;
    }

}

```

However, there is a bunch of wiring need to get done in order to make that work.  Thankfully, if you want to just use what I've written, you can use that and not have to worry about the writing.  Out of the box, MCM provides a configuration factory.  It will build the two classes that are needed: the manager **(provides the appropriate configuration object to the application)**, and the builder **(builds and populates the configuration object)**.

#### Configuration Files

But before you can do any of that you need to provide a minimum of 3 (!) configuration files.  The are

1. The global magium configuration file.  This contains a preset list of configuration files, cache adapter, and database adapter configurations.  It is typically called `magium-configuration.xml` and generally needs to be in the root of your project, preferably just below the `vendor` directory.
2. The context configuration file.  This contains the context hierarchy.  It is usually called `contexts.xml` and should be in a location refenced in the `magium-configuration.xml` file.
3. At least one actual configuration file.

1 and 2 are actually really easy to get started.  You can write them by hand or you can use the handy dandy `vendor/bin/magium-configuration` CLI script.  The first time you run that it will ask you where to put both files.

```
php vendor/bin/magium-configuration
Could not find a magium-configuration.xml file.  Where would you like me to put it?
  [0] C:\Projects\magium-configuration.xml
  [1] C:\Projects\example\magium-configuration.xml
 > 1
Wrote XML configuration file to: C:\Projects\example\magium-configuration.xml
The context file C:\Projects\example\contexts.xml does not exist next to the magium-configuration.xml file.  Create it? y
```

Open them up and take a look around.

> A quick word on XML.  Why is he using XML?  It's not cool anymore!  YAML or JSON would make me feel much better.
>
> The answer is that neither YAML, JSON, nor PHP files are self documenting.  I'm sure there are tools out there that might allow you to do some of that but there is nothing out there that compares to what is available for XML.  I am a firm believer that some for of introspection which can be used for error detection or, more importantly, code completion, significantly reduce the amount of startup time.  It simply requires less memorizing and less copy-and-pasting.  But that said, if you look in the /lib directory you will find that the intention is to support all of those formats for the masochists out there.  I just haven't done it yet.

Take a look inside the `magium-configuration.xml` file.  Much of it is, I believe, self explanatory.  However, note the `persistenceConfiguration` and `cache` nodes.  Those are converted to arrays and passed into the `Zend\Db\Adapter\Adapter` and `Zend\Cache\StorageFactory` classes, respectively.  Note, also, the `contextConfigurationFile` node.  That contains a reference to the next configuration file.

The `contexts.xml` file does not need to be called that; that's just what the CLI creates by default.  It contains a hierarchical list of contexts.  Each context must be uniquely named, but they can be hierarchical.

```
<?xml version="1.0" encoding="UTF-8" ?>
<magiumDefaultContext xmlns="http://www.magiumlib.com/ConfigurationContext">
    <context id="production" title="Production">
        <context id="website1" title="Website 1"/>
        <context id="website2" title="Website 2"/>
    </context>
    <context id="development" title="Development" />
</magiumDefaultContext>
```

There is always a default context that all contexts inherit from.  If you don't want contexts, just have the `<magiumDefaultContext />` node alone in there.

The structure allows for an overriding hierarchy.  That means that if there are two different values specified for `website1` and `website2` they will have two different values in the resulting config object.  However, if there is a value provided for the context `production` it will exist in both `website1` and `website2` unless overridden in those contexts.  Nice and easy.
 
 The next configuration file are the actual settings file.  There must be at least one, but there can also be many.  Hundreds, if you want.  They are all merged into one big meta configuration XML file which is used to denote the paths to retrieve the configuration settings.

For example, take two configuration files:

```
<magiumConfiguration xmlns="http://www.magiumlib.com/Configuration">
    <section identifier="web">
        <group identifier="items">
            <element identifier="element1" />
        </group>
    </section>
</magiumConfiguration>
```

and 

```
<magiumConfiguration xmlns="http://www.magiumlib.com/Configuration">
    <section identifier="web">
        <group identifier="items">
            <element identifier="element2" />
        </group>
    </section>
</magiumConfiguration>
```

They will merge together to create:

```
<magiumConfiguration xmlns="http://www.magiumlib.com/Configuration">
    <section identifier="web">
        <group identifier="items">
            <element identifier="element1" />
            <element identifier="element2" />
        </group>
    </section>
</magiumConfiguration>
```

The ID attributes are used to denote the actual paths that will be queried.  So, for this configuration, you can ask `Config` object for the values for `web/items/element1` and `web/items/element2`.

Also, you can provide default values and descriptions:

```
<magiumConfiguration xmlns="http://www.magiumlib.com/Configuration">
    <section identifier="section">
        <group identifier="group">
            <element identifier="element1">
                <description>Just some silly value</description>
                <value>Test Value</value>
            </element>
        </group>
    </section>
</magiumConfiguration>
```

#### Configuring Your Application

If you are going to use the basic factory included with the package, wiring MCM is pretty easy.

```
$factory = new \Magium\Configuration\MagiumConfigurationFactory();
$config = $factory->getManager()->getConfiguration(getenv('ENVIRONMENT'));
```

That's it.  Then, wherever you are using that configuration object and (presuming that you are using the previous XML files) retrieving a configuration value is as easy as:

```
echo config->getValue('web/items/element1');
```

More examples, both raw and for various frameworks, will be found at the [Configuration Manager Examples](https://github.com/magium/magium-configuration-manager-examples) page.

### Setting Configuration Values

Configuration values are set via the command line using the `vendor/bin/magium-configuration` script using the `magium:configuration:set` command.

For example, considering the previous configuration file, you can set the value by calling:

```
php vendor/bin/magium-configuration magium:configuration:set section/group/element1 "some value"
```

Then when you call 

```
$config->getValue('web/items/element1')
```

You will get the configured value.

Additionally, if you want to set the value for a particular context you append it to the end of the `set` command:

```
php vendor/bin/magium-configuration magium:configuration:set section/group/element1 "some value" website1
```

And that's about it.

### Third Party Integration

If you are building any third party modules that need to be hooked into the MCM the easiest way is to hook in with `Magium\Configuration\File\Configuration\ConfigurationFileRepository`.  It is used when building the configuration object, prior to storing it in the cache.  Do this via Composer autoloaders.

```
{
    "autoload": {
        "files": [
            "registration.php"
        ]
    }
}
```
composer.json

```

$instance = \Magium\Configuration\File\Configuration\ConfigurationFileRepository::getInstance();
$instance->addSecureBase(realpath('etc'));
$instance->registerConfigurationFile(
    new \Magium\Configuration\File\Configuration\XmlFile(
        realpath(
            'etc/settings.xml'
        )
    )
);


```
registration.php

Your configuration settings are now included with your customer's application.  Note, however, that you should not have a `magium-configuration.xml` file or your own contexts file.  That is for the primary application to manage.

### Moving forward

Here are a list of features that I want to have included

* ~~An HTML based UI for making configuration changes~~
  * Embeddable user restrictions
  * ACL resources
* ~~A local/remote cache system that caches the configuration locally, but synchronizes with a remote cache~~
* ~~Integrations/adapters with multiple third party Dependency Injection Containers, allowing you to either manage DI configuration centrally, and/or, modify DI configuration on the fly without requiring a deployment.~~ (kind of)
* ~~Oh yes, and integration with persistent storage~~
