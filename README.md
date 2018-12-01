![](img/logo.png)
# JSON Machine

JSON Machine is a fast, efficient and easy-to-use JSON stream parser based on coroutines
developed for unpredictably long JSON streams or documents. Main features are:

- Ease of use. Just iterate JSON of any size with `foreach`. No events and callbacks.
- Constant memory footprint for unpredictably large JSON documents.
- Speed. Performace critical code contains no unnecessary function calls, no regular expressions
and uses native `json_decode` to decode JSON document chunks.
- Supports efficient iteration on any subtree of the document, specified by [Json Pointer](#json-pointer)

## TL;DR;
JSON Machine enables you to drop-in replace non efficient way of iterating big JSONs:

```php
<?php
$users = json_decode(file_get_contents('500MB-users.json'));
foreach ($users as $id => $user) {
    // the script will probably die before getting here
}
```

... by the efficient way:

```php
<?php
$users = \JsonMachine\JsonMachine::fromFile('500MB-users.json');
foreach ($users as $id => $user) {
    // process $user with minimal memory footprint
}
```

Random access like `$users[42]` **is not possible**. However you can scan the array in `foreach` and find the item.

## Parsing JSON documents

### Simple document
Let's say that `big.json` contains this really big JSON document:
```json
// big.json
{
    "apple": {
        "color": "red"
    },
    "pear": {
        "color": "yellow"
    }
}
``` 
It can be parsed this way:
```php
<?php

$jsonStream = \JsonMachine\JsonMachine::fromFile('big.json');

foreach ($jsonStream as $name => $data) {
    // 1st iteration: $name === "apple" and $data === ["color" => "red"]
    // 2nd iteration: $name === "pear" and $data === ["color" => "yellow"]
}
```

Parsing an array instead of a dictionary follows the same logic.
The key in a foreach will be a numeric index of an item.

### Parsing a subtree
If you want to iterate only `fruits-key` subtree in this `fruits.json`:
```json
// fruits.json
{
    "fruits-key": {
        "apple": {
            "color": "red"
        },
        "pear": {
            "color": "yellow"
        }
    }
}
```
do it like this:
```php
<?php

$jsonStream = \JsonMachine\JsonMachine::fromFile("fruits.json", "/fruits-key" /* <- Json Pointer */);
foreach ($jsonStream as $name => $data) {
    // The same as above, which means:
    // 1st iteration: $name === "apple" and $data === ["color" => "red"]
    // 2nd iteration: $name === "pear" and $data === ["color" => "yellow"]
}
```

> Note:
>
> Value of `fruits-key` is not loaded into memory at once, but only one item in
> `fruits-key` at a time. It is always one item in memory at a time at the level/subtree
> you are currently iterating. Thus the memory consumption is constant.

<a name="json-pointer"></a>
#### Few words about Json Pointer
It's a way of addressing one item in JSON document. See the [Json Pointer RFC 6901](https://tools.ietf.org/html/rfc6901).
It's very handy, because sometimes the JSON structure goes deeper, and you want to iterate a subtree,
not the main level. So you just specify the pointer to the JSON array or object you want to iterate and off you go.
When the parser hits the collection you specified, iteration begins. It is always a second parameter in all
`JsonMachine::from*` functions. If you specify pointer to scalar value (which logically cannot be iterated)
or non existent position in the document, an exception is thrown.

Some examples:

| Json Pointer value | Will iterate through                                                                              |
|--------------------|---------------------------------------------------------------------------------------------------|
| empty string       | `["this", "array"]` or `{"a": "this", "b": "dictionary"}` will be iterated (main level - default) |
| `/result/items`    | `{"result":{"items":["this","array","will","be","iterated"]}}`                                    |
| `/0/items`         | `[{"items":["this","array","will","be","iterated"]}]` (supports array indexes)                    |
| `/` (gotcha!)      | `{"":{"items":["this","array","will","be","iterated"]}}` (no kidding, see the spec)               |

  
## Parsing API responses
API response or any other JSON stream is parsed exactly the same way as file is. The only difference
is, you use `JsonMachine::fromStream($streamResource)` for it, where `$streamResource` is the stream
resource with the JSON document. The rest is the same as with parsing files.

### GuzzleHttp
Guzzle uses its own streams, but they can be converted back to PHP streams by calling
`\GuzzleHttp\Psr7\StreamWrapper::getResource()`. Pass the result of this function to
`JsonMachine::fromStream` function and you're set up. See working
[GuzzleHttp example](src/examples/guzzleHttp.php).

## Efficiency of parsing streams/files
JSON Machine reads the stream or file 1 JSON item at a time and generates corresponding 1 PHP array at a time.
This is the most efficient way, because if you had say 10,000 users in JSON file and wanted to parse it using
`json_decode(file_get_contents('big.json'))`, you'd have the whole string in memory as well as all the 10,000
PHP structures. Following table demonstrates a concept of the difference:

|                            | String items in memory at a time | Decoded PHP items in memory at a time | Total |
|----------------------------|---------------------------------:|--------------------------------------:|------:|
| `json_decode`              |                            10000 |                                 10000 | 20000 |
| `JsonMachine::fromStream`  |                                1 |                                     1 |     2 |

This means, that `JsonMachine::fromStream` is constantly efficient for any size of processed JSON. 100 GB no problem.

## Efficiency of parsing in-memory JSON strings
There is also a method `JsonMachine::fromString()`. You may wonder, why is it there. Why just not use
`json_decode`? True, when parsing short strings, JSON Machine may be overhead. But if you are
forced to parse a big string and the stream is not available, JSON Machine may be better than `json_decode`.
The reason is that unlike `json_decode` it still traverses the JSON string one item at a time and doesn't
load the whole resulting PHP structure into memory at once.

Let's continue with the example with 10,000 users. This time they are all in string in memory.
When decoding that string with `json_decode`, 10,000 arrays (objects) is created in memory and then the result
is returned. JSON Machine on the other hand creates single array for found item in the string and yields it back
to you. When you process this item and iterate to the next one, another single array is created. This is the same
behaviour as with streams/files. Following table puts the concept into perspective:

|                            | String items in memory at a time | Decoded PHP items in memory at a time | Total |
|----------------------------|---------------------------------:|--------------------------------------:|------:|
| `json_decode`              |                            10000 |                                 10000 | 20000 |
| `JsonMachine::fromString`  |                            10000 |                                     1 | 10001 |

This is still about twice as efficient than `json_decode`.

## Error handling
When any part of the JSON stream is malformed, `SyntaxError` exception is thrown. Better solution is on the way.

## Running tests
```bash
composer install
vendor/bin/phpunit
```
To run tests on all supported PHP platforms install docker to your machine and run `tests/docker-run-all-platforms.sh`

## Installation
```bash
composer require halaxa/json-machine
```
or clone or download this repository (not recommended).

## License
Apache 2.0

Cogwheel element: Icons made by [TutsPlus](https://www.flaticon.com/authors/tutsplus)
from [www.flaticon.com](https://www.flaticon.com/)
is licensed by [CC 3.0 BY](http://creativecommons.org/licenses/by/3.0/)
