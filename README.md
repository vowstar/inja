[<div align="center"><img width="500" src="https://raw.githubusercontent.com/pantor/inja/master/doc/logo.jpg"></div>](https://github.com/pantor/inja/releases)


[![Build Status](https://travis-ci.org/pantor/inja.svg?branch=master)](https://travis-ci.org/pantor/inja)
[![Build status](https://ci.appveyor.com/api/projects/status/qtgniyyg6fn8ich8?svg=true)](https://ci.appveyor.com/project/pantor/inja)
[![Coverage Status](https://img.shields.io/coveralls/pantor/inja.svg)](https://coveralls.io/r/pantor/inja)
[![Codacy Status](https://api.codacy.com/project/badge/Grade/aa2041f1e6e648ae83945d29cfa0da17)](https://www.codacy.com/app/pantor/inja?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=pantor/inja&amp;utm_campaign=Badge_Grade)
[![Github Releases](https://img.shields.io/github/release/pantor/inja.svg)](https://github.com/pantor/inja/releases)
[![Github Issues](https://img.shields.io/github/issues/pantor/inja.svg)](http://github.com/pantor/inja/issues)
[![GitHub License](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/pantor/inja/master/LICENSE)


Inja is a template engine for modern C++, loosely inspired by [jinja](http://jinja.pocoo.org) for python. It has an easy and yet powerful template syntax with all variables, loops, conditions, includes, callbacks, comments you need, nested and combined as you like. Inja uses the wonderful [json](https://github.com/nlohmann/json) library by nlohmann for data input and handling. Most importantly, *inja* needs only two header files, which is (nearly) as trivial as integration in C++ can get. Of course, everything is tested on all relevant compilers. Have a look what it looks like:

```c++
json data;
data["name"] = "world";

inja::render("Hello {{ name }}!", data); // Returns "Hello world!"
```


## Integration

Inja is a headers only library, which can be downloaded in the releases or directly from the `src/` folder. Inja uses json by nlohmann as its single dependency, so make sure that it is included before inja. json can be found [here](https://github.com/nlohmann/json/releases).

```c++
#include "json.hpp"
#include "inja.hpp"

// For convenience
using namespace inja;
using json = nlohmann::json;
```

You can also integrate inja in your project using [Hunter](https://github.com/ruslo/hunter), a package manager for C++.


## Tutorial

This tutorial will give you an idea how to use inja. It will explain the most important concepts and give practical advices using examples and executable code.

### Template Rendering

The basic template rendering takes a template as a `std::string` and a `json` object for all data. It returns the rendered template as an `std::string`.

```c++
json data;
data["name"] = "world";

render("Hello {{ name }}!", data); // "Hello world!"

// For more advanced usage, an environment is recommended
Environment env = Environment();

// Render a string with json data
std::string result = env.render("Hello {{ name }}!", data); // "Hello world!"

// Or directly read a template file
Template temp = env.parse_template("./template.txt");
std::string result = temp.render(data); // "Hello world!"

data["name"] = "Inja";
std::string result = temp.render(data); // "Hello Inja!"

// Or read a json file for data directly from the environment
result = env.render_template("./template.txt", "./data.json");

// Or write a rendered template file
temp.write(data, "./result.txt")
env.write("./template.txt", "./data.json", "./result.txt")
```

The environment class can be configured to your needs.
```c++
// With default settings
Environment env_default = Environment();

// With global path to template files
Environment env = Environment("../path/templates/");

// With global path where to save rendered files
Environment env = Environment("../path/templates/", "../path/results/");

// Choose between JSON pointer or dot notation to access elements
env.set_element_notation(ElementNotation::Pointer); // (default) e.g. time/start
env.set_element_notation(ElementNotation::Dot); // e.g. time.start

// With other opening and closing strings (here the defaults, as regex)
env.set_variables("\\{\\{", "\\}\\}"); // Variables {{ }}
env.set_comments("\\{#", "#\\}"); // Comments {# #}
env.set_statements("\\{\\%", "\\%\\}"); // Statements {% %} for many things, see below
env.set_line_statements("##"); // Line statement ## (just an opener)
```

### Variables

Variables are rendered within the `{{ ... }}` expressions.
```c++
json data;
data["neighbour"] = "Peter";
data["guests"] = {"Jeff", "Tom", "Patrick"};
data["time"]["start"] = 16;
data["time"]["end"] = 22;

// Indexing in array
render("{{ guests/1 }}", data); // "Tom"

// Objects
render("{{ time/start }} to {{ time/end }}pm"); // "16 to 22pm"
```
In general, the variables can be fetched using the [JSON Pointer](https://tools.ietf.org/html/rfc6901) syntax. For convenience, the leading `/` can be ommited. If no variable is found, valid JSON is printed directly, otherwise an error is thrown.


### Statements

Statements can be written either with the `{% ... %}` syntax or the `##` syntax for entire lines. The most important statements are loops, conditions and file includes. All statements can be nested.

#### Loops

```c++
// Combining loops and line statements
render(R"(Guest List:
## for guest in guests
	{{ index1 }}: {{ guest }}
## endfor )", data)

/* Guest List:
	1: Jeff
	2: Tom
	3: Patrick */
```
In a loop, the special variables `index (number)`, `index1 (number)`, `is_first (boolean)` and `is_last (boolean)` are available. You can also iterate over objects like `{% for key, value in time %}`.

#### Conditions

Conditions support the typical if, else if and else statements. Following conditions are for example possible:
```c++
// Standard comparisons with variable
render("{% if time/hour >= 18 %}…{% endif %}", data); // True

// Variable in list
render("{% if neighbour in guests %}…{% endif %}", data); // True

// Logical operations
render("{% if guest_count < 5 and all_tired %}…{% endif %}", data); // True

// Negations
render("{% if not guest_count %}…{% endif %}", data); // True
```

#### Includes

This includes other template files, relative from the current file location.
```
{% include "footer.html" %}
```

### Functions

A few functions are implemented within the inja template syntax. They can be called with
```c++
// Upper and lower function, for string cases
render("Hello {{ upper(neighbour) }}!", data); // "Hello PETER!"
render("Hello {{ lower(neighbour) }}!", data); // "Hello peter!"

// Range function, useful for loops
render("{% for i in range(4) %}{{ index1 }}{% endfor %}", data); // "1234"

// Length function (please don't combine with range, use list directly...)
render("I count {{ length(guests) }} guests.", data); // "I count 3 guests."

// Get first and last element in a list
render("{{ first(guests) }} was first.", data); // "Jeff was first."
render("{{ last(guests) }} was last.", data); // "Patir was last."

// Sort a list
render("{{ sort([3,2,1]) }}", data); // "[1,2,3]"
render("{{ sort(guests) }}", data); // "[\"Jeff\", \"Patrick\", \"Tom\"]"


// Round numbers to a given precision
render("{{ round(3.1415, 0) }}", data); // 3
render("{{ round(3.1415, 3) }}", data); // 3.142

// Check if a value is odd, even or divisible by a number
render("{{ odd(42) }}", data); // false
render("{{ even(42) }}", data); // true
render("{{ divisibleBy(42, 7) }}", data); // true

// Maximum and minimum values from a list
render("{{ max([1, 2, 3]) }}", data); // 3
render("{{ min([-2.4, -1.2, 4.5]) }}", data); // -2.4

// Set default values if variables are not defined
render("Hello {{ default(neighbour, \"my friend\") }}!"); // "Hello Peter!"
render("Hello {{ default(colleague, \"my friend\") }}!"); // "Hello my friend!"
```

### Callbacks

You can create your own and more complex functions with callbacks.
```c++
Environment env = Environment();

/*
 * Callbacks are defined by its:
 * - name, which is equal to the function name
 * - number of arguments
 * - callback function. Implemented with std::function, you can for example use lambdas.
 */
env.add_callback("double", 1, [&env](Parsed::Arguments args, json data) {
	int number = env.get_argument<int>(args, 0, data); // Adapt the type and index of the argument
	return 2 * number;
});

// You can then use a callback like a regular function
env.render("{{ double(16) }}", data); // "32"

// A callback without argument can be used like a dynamic variable:
std::string greet = "Hello";
env.add_callback("double-greetings", 0, [greet](Parsed::Arguments args, json data) {
	return greet + " " + greet + "!";
});
env.render("{{ double-greetings }}", data); // "Hello Hello!"
```

### Comments

Comments can be written with the `{# ... #}` syntax.
```c++
render("Hello{# Todo #}!", data); // "Hello!"
```



## Supported compilers

Currently, the following compilers are tested:

- GCC 4.9 - 7.1 (and possibly later)
- Clang 3.6 - 3.7 (and possibly later)
- Microsoft Visual C++ 2015 / Build Tools 14.0.25123.0 (and possibly later)
- Microsoft Visual C++ 2017 / Build Tools 15.1.548.43366 (and possibly later)



## License

Inja is licensed under the [MIT License](https://raw.githubusercontent.com/pantor/inja/master/LICENSE).
