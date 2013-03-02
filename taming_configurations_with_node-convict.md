Taming Configurations with node-convict
=====

In this installment of "A Node.JS Holiday Season" series we'll take a look at [`node-convict`](https://github.com/lloyd/node-convict), a tool to help manage the configuration of node.js applications.

There are two main concerns regarding application configuration:

* Most applications will have at least a few different deployment environments, each with their own configuration needs.
* Including credentials and sensitive information in source can be problematic.

These concerns can be addressed by initializing certain settings based on the environment, and using environmental variables for more sensitive settings. A common pattern used by node.js developers is to create a module that exports the configuration, e.g.:

    var conf = {
      // the application environment
      // "production", "development", or "test
      env: process.env.NODE_ENV || "development",

      // the IP address to bind
      ip: process.env.IP_ADDRESS || "127.0.0.1",

      // the port to bind
      port: process.env.PORT || 0

      // database settings
      database: {
        host: process.env.DB_HOST || "localhost:8091",
        user: process.env.DB_USER || "Administrator",
        password: process.env.DB_PASSWORD || "password"
      }
    };

    module.exports = conf;

This works well enough, but there are additional concerns:

* **What if a setting was configured incorrectly?** We can save headaches by detecting and reporting misconfigurations as early as possible.
* **How easily is it understood** by Ops/QA/and other collaborators that may need to adjust settings or diagnose issues? A more declarative format that also embeds documentation can make lives easier.

Convict addresses these additional concerns by introducing a **configuration schema** where you can set **type information, default values, environmental variables, and documentation** for each setting.

# Enter convict

With convict, the example from above becomes:

    var convict = require('convict');

    var conf = convict({
      env: {
        doc: "The applicaton environment.",
        format: ["production", "development", "test"],
        default: "development",
        env: "NODE_ENV"
      },
      ip: {
        doc: "The IP address to bind.",
        format: "ipaddress",
        default: "127.0.0.1",
        env: "IP_ADDRESS"
      },
      port: {
        doc: "The port to bind.",
        format: "port",
        default: 0,
        env: "PORT"
      },
      database: {
        host: {
          default: "localhost:8091",
          env: "DB_HOST"
        },
        user: {
          default: "Administrator",
          env: "DB_USER"
        },
        password: {
          default: "password",
          env: "DB_PASSWORD"
        }
      }
    });

    conf.validate();

    module.exports = conf;

The information is more or less the same, but encoded in the schema. Since all of the information is encoded in the schema, we can export it and display it in an easier to read format, and we can use it for validation. This declarative approach is what helps convict be more robust and collaborator friendly. 

# What's in a schema
 You'll notice four possible properties for each setting – each aiding in our goal of a more robust and easily digestible configuration.

* **Type information**: the `format` property specifies either a built-in convict format (`ipaddress`, `port`, `int`, etc.), or it can be a function to check a custom format. During validation, if a format check fails it will be added to the error report.
* **Default values**:  Every setting *must* have a default value.
* **Environmental variables**: If the variable specified by `env` has a value, it will overwrite the setting's default value.
* **Documentation**: The `doc` property is pretty self-explanatory. The nice part about having it in the schema rather than as a comment is that we can call `conf.toSchemaString()` and have it displayed in the output.

# Layering additional configurations
The set of defaults becomes a base configuration on top of which you can overlay additional configurations, using `conf.load` and `conf.loadFile`. For example, you could conditionally overlay a JavaScript object with settings for a specific environment:


    var conf = convict({
      // snip ... assume the same schema as above
    });

    if (conf.get('env') === 'production') {
      // use the production port and database host
      conf.load({
        port: 8080
        database: {
          host: "ec2-117-21-174-242.compute-1.amazonaws.com:8091"
        }
      });
    }

    conf.validate();

    module.exports = conf;


Alternatively, if you create a separate configuration file for each environment, you could simply overlay them with `loadFile`:

    conf.loadFile('./config/' + conf.get('env') + '.json');

Layering configurations with `load` and `loadFile` is useful if you have common settings for specific environments that don't *need* to be set in environmental variables. Having separate, declarative JSON configurations can provide greater visibility of which settings should change between environments. And the files are loaded with [cjson](https://github.com/kof/node-cjson), so they may contain as many comments as you want for additional clarification. Also note that **environmental variables always take precedent**, even after settings have been loaded using `load` or `loadFile`. To see what the current settings look like, you can serialize the whole thing using `conf.toString()`.

# More on formats

Once the settings have been composed, you can check that each setting's value conforms to the correct format, as defined in the schema. Convict provides a few built-in formats such as `"url"`, `"ports"`, and `"ipaddress"`, among others, and you may use one of JavaScript's global constructors (e.g. `Number`) to designate a type. If you leave the `format` property out entirely, convict will check that the setting has the same type as the default value (according to [`Object.prototype.toString.call`](http://perfectionkills.com/instanceof-considered-harmful-or-how-to-write-a-robust-isarray/)). For instance, the following three schemas are equivalent:

    var conf1 = convict({
        name: {
          format: String
          default: 'Brendan'
        }
      });

    // with no format specified, convict will assume it's the type
    // of the default value
    var conf2 = convict({
        name: {
          default: 'Brendan'
        }
      });

    // a more succinct version
    var conf3 = convict({
        name: 'Brendan'
      });

In place of a built-in type, you can also provide your own format checking function. For example:

    var check = require('validator').check;

    var conf = convict({
        key: {
          doc: "API key",
          format: function (val) {
            check(val, 'should be a 64 character hex key').regex(/[a-fA-F0-9]{64}/);
          },
          default: '3cec609c9bc601c047af917a544645c50caf8cd606806b4e0a23312441014deb'
        }
      });

Additionally, there is an enum-style format as seen in the examples, where you can specify an explicit set of acceptable values, e.g. `["production", "development", "test"]`. Any value not in the list will fail validation.

## V for Validation
Calling `conf.validate()` will throw an error with details on each setting that failed to validate, if there were any. This is important for avoiding redeployment after each individual configuration error. Using the configuration  from the custom key format example, here's what an error would look like:

    conf.set('key', 'foo');
    conf.validate();
    // throws Error: key: should be a 64 character hex key: value was "foo"


# Conclusion
Convict expounds on the standard pattern of configuring node.js applications in a way that is more robust and accessible to collaborators, who may have less interest in digging through imperative code in order to inspect or modify settings. With the configuration schema, we can give project collaborators more **context** on each setting as well as provide **validation and early failures** when configuration goes wrong.
