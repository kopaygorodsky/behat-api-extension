# Behat API Extension

[![Current build Status](https://secure.travis-ci.org/imbo/behat-api-extension.png)](http://travis-ci.org/imbo/behat-api-extension)

**This extension is a work in progress. The steps mentioned below will probably change along the way until a stable release has been made, so expect things to break without warning. Not all steps in the README have been fully implemented yet either.**

This Behat extension provides an easy way to test JSON-based API's in [Behat 3](http://behat.org). Inspired by [behat/web-api-extension](https://github.com/Behat/WebApiExtension/) and originally written to test the [Imbo API](http://imbo.io).

## Requirements

Behat API extension requires:

* [PHP](http://php.net) 5.6+
* [behat/behat](http://behat.org) 3.0+
* [guzzlehttp/guzzle](http://guzzlephp.org) 6.0+
* [beberlei/assert](https://github.com/beberlei/assert/) 2.1+

## Installation

Install the extension by adding the following to your `composer.json` file:

    {
        "require-dev": {
            "imbo/behat-api-extension": "^1.0@dev"
        }
    }

and then updating your dependencies by issuing `composer update imbo/behat-api-extension`.

After you have installed it you need to activate the extension in your `behat.yml` file:

    default:
        suites:
            default:
                # ...

        extensions:
            Imbo\BehatApiExtension: ~

## Configuration

The following configuration options are available:

| Key      | Type   | Default value         | Description                            |
| -------- | ------ | --------------------- | -------------------------------------- |
| base_uri | string | http://localhost:8080 | Base URI of the application under test |

## Usage

The extension allows you to use the following steps in your features:

### Set up the request

The following steps can be used prior to sending a request.

#### Given I am authenticating as `:username` with password `:password`

Use this step when the URL you are requesting requires basic auth.

**Examples:**

| Step                                                               | :username | :password |
| ------------------------------------------------------------------ | --------- | --------- |
| Given I am authenticating as `foo` with password `bar`             | `foo`     | `bar`     |
| Given I am authenticating as "`foo bar`" with password '`bar foo`' | `foo bar` | `bar foo` |
| Given I am authenticating as '`"foo"`' with password "`'bar'`"     | `"foo"`   | `'bar'`   |

#### Given the `:header` request header is `:value`

Set the `:header` request header to `:value`. Can be repeated to set multiple headers or to set the same header multiple times.

Trying to force specific headers to have certain values combined with other steps that ends up modifying request headers (for instance attaching files) can lead to undefined behaviour.

**Examples:**

| Step                                                                                       | :header      | :value            |
| ------------------------------------------------------------------------------------------ | ------------ | ----------------- |
| Given the "`User-Agent`" request header is "`test/1.0`"                                    | `User-Agent` | `test/1.0`        |
| Given the "`X-Foo`" request header is `Bar`<br>Given the "`X-Foo`" request header is `Baz` | `X-Foo`      | `Bar, Baz`        |
| Given the `Accept` request header is "`application/json`"                                  | `Accept`     | `application/json`|

#### Given I attach `:path` to the request as `:name`

Attach a file to the request (causing a `multipart/form-data` request, populating the `$_FILES` array on the server). Can be repeated to attach several files. If a specified file does not exist an `InvalidArgumentException` exception will be thrown. `:path` is relative to the working directory unless it's absolute.

**Examples:**

| Step                                                                  | :path                   | Entry in `$_FILES` on the server (:name) |
| --------------------------------------------------------------------- | ----------------------- | ---------------------------------------- |
| Given I attach "`/path/to/file.jpg`" to the request as `file1`        | `/path/to/file.jpg`     | $\_FILES['`file1`']                      |
| Given I attach "`c:\some\file.jpg`" to the request as `file2`         | `c:\some\file.jpg`      | $\_FILES['`file2`']                      |
| Given I attach "`features/some.feature`" to the request as `feature`  | `features/some.feature` | $\_FILES['`feature`']                    |

### Send the request

After setting up the request it can be sent to the server in a few different ways. Keep in mind that all configuration regarding the request must be set prior to any of these steps, as they will actually send the request.

#### When I request `:path` (using HTTP `:method`)

`:path` is relative to the `base_uri` configuration option, and `:method` is any HTTP method, for instance `POST` or `DELETE`. If the last part of the step is omitted, `HTTP GET` will be used. If the `:path` starts with a slash, it will be relative to the root of `base_uri`.

**Examples:**

*Assume that the `base_uri` configuration option has been set to `http://example.com/dir` in the following examples.*

| Step                                               | :path               | :method  | Resulting URL                         |
| -------------------------------------------------- | ------------------- | -------- | ------------------------------------- |
| When I request "`/?foo=bar&bar=foo`"               | `/?foo=bar&bar=foo` | `GET`    | `http://example.com/?foo=bar&bar=foo` |
| When I request "`/some/path`" using HTTP `DELETE`  | `/some/path`        | `DELETE` | `http://example.com/some/path`        |
| When I request `foobar` using HTTP `POST`          | `foobar`            | `POST`   | `http://example.com/dir/foobar`       |

#### When I request `:path` using HTTP `:method` with (`JSON`) body: `<PyStringNode>`

This step can be used to attach a body to the request. The same as above applies for `:path` and `:method`. If the `JSON` part is added to the step the `Content-Type` request header will be set to `application/json` (regardless of whether or not the `Content-Type` header has already been set with the `Given the :header request header is :value` step described above).

**Examples:**

    When I request "some/endpoint" using HTTP POST with body:
        """
        some POST body
        """

and with the optional `JSON` identifier, that sets the `Content-Type` request header to `application/json`:

    When I request "some/endpoint" using HTTP PUT with JSON body:
        """
        {"foo": "bar"}
        """

The extension will validate the JSON data before sending the request using this step, and if it's not valid an `InvalidArgumentException` exception will be thrown. If you want to send invalid JSON data to the server, you can do the following:

    Given the "Content-Type" request header is "application/json"
    When I request "some/endpoint" using HTTP POST with body:
        """
        {"some":"invalid":"json"}
        """

#### When I send `:filePath` (as `:mimeType`) to `:path` using HTTP `:method`

Send the file at `:filePath` to `:path` using the `:method` HTTP method. `:filePath` is relative to the working directory unless it's absolute. `:method` would typically be `PUT` or `POST` for this action, but any valid HTTP method can be used. The optional `:mimeType` can be added to force the `Content-Type` request header. If not specified the extension will try to guess the mime type using available methods.

**Examples:**

| Step                                                                                 | :filePath        | :mimeType                     | :path       | :method |
| ------------------------------------------------------------------------------------ | ---------------- | ----------------------------- | ----------- | ------- |
| When I send "`/some/file.jpg`" to "`/endpoint`" using HTTP `POST`                    | `/some/file.jpg` | `image/jpeg` (guessed)        | `/endpoint` | `POST`  |
| When I send "`file.bar`" as "`application/foobar`" to "`/endpoint`" using HTTP `PUT` | `file.bar`       | `application/foobar` (forced) | `/endpoint` | `PUT`   |

### Verify server response

After a request has been sent, some steps exist that can be used to verify the response from the server. All steps that matches response content assumes JSON-data in the response body unless noted otherwise.

#### Then the response code should (`not`) be `:code`

Match the response code to `:code`. If the optional `not` is added, the response should **not** match the response code.

**Examples:**

| Step                                         | :code | Matches `200` | Matches `304` | Matches `404` |
| -------------------------------------------- | ----- | ------------- | ------------- | ------------- |
| Then the response code should be `200`       | `200` | Yes           | No            | No            |
| Then the response code should be `404`       | `404` | No            | No            | Yes           |
| Then the response code should `not` be `304` | `304` | Yes           | No            | Yes           |

#### Then the response is (`not`) `:group`

Match the response code to a group. If the optional `not` is added, the response should **not** be in the specified group.

Allowed groups and their ranges are:

| Group           | Response code range |
| --------------- | ------------------- |
| `informational` | 100 to 199          |
| `success`       | 200 to 299          |
| `redirection`   | 300 to 399          |
| `client error`  | 400 to 499          |
| `server error`  | 500 to 599          |

**Examples:**

| Step                                        | :group          | Response code range that matches |
| ------------------------------------------- | --------------- | -------------------------------- |
| Then the response is `informational`        | `informational` | 100 to 199                       |
| Then the response is "`client error`"       | `client error`  | 400 to 499                       |
| Then the response is `not` "`client error`" | `client error`  | 100 to 399 and 500 to 599        |

#### Then the `:header` response header is (`not`) present

This step can be used to assert that the `:header` response header is present, or not (if used with the optional `not` keyword). The value of `:header` is case-insensitive.

**Examples:**

*Assume that these response headers exist in the following examples:*

* *Content-Encoding: gzip*
* *Content-Type: application/json*
* *Content-Length: 186*
* *Date: Wed, 31 Aug 2016 15:06:02 GMT*

| Step                                                         | :header          | Test passes? |
| ------------------------------------------------------------ | ---------------- | ------------ |
| Then the `Vary` response header is present                   | `Vary`           | No           |
| Then the `vary` response header is `not` present             | `Vary`           | Yes          |
| Then the "`Content-Length`" response header is present       | `Content-Length` | Yes          |
| Then the "`content-length`" response header is `not` present | `content-length` | No           |

#### Then the `:header` response header (`is`|`matches`) `:value`

This step can be used to verify the value of one or more response headers.

The step supports two different comparison modes, `is` and `matches`. `is` will compare the values using regular string comparison (`==`), and when `matches` is used, the `:value` must be a valid regular expression, complete with delimiters and optional modifiers. The expression will be fed straight into [preg_match](http://php.net/preg_match), so make sure it's valid before using it to verify values.

**Examples:**

*Assume that these response headers exist in the following examples:*

* *Content-Length: 14327*
* *X-Foo: foo, bar*

| Step                                                                         | :header          | :value                          | Mode               | Matches header |
| ---------------------------------------------------------------------------- | ---------------- | ------------------------------- | ------------------ | -------------- |
| Then the "`Content-Length`" response header is `15000`                       | `Content-Length` | `15000`                         | Comparison         | No             |
| Then the "`content-length`" response header matches "`/[0-9]+/`"             | `content-length` | `/[0-9]+/`                      | Regular expression | Yes            |
| Then the "`x-foo`" response header matches "<code>/(FOO&#124;BAR)/i</code>"  | `x-foo`          | <code>/(FOO&#124;BAR)/i</code>  | Regular expression | Yes            |
| Then the "`X-FOO`" response header matches "<code>/^(foo&#124;bar)$/</code>" | `X-FOO`          | <code>/^(foo&#124;bar)$/</code> | Regular expression | No             |
| Then the "`X-foo`" response header is "`foo, bar`"                           | `X-foo`          | `foo, bar`                      | Comparison         | Yes            |

For more information regarding regular expressions and the usage of modifiers, [refer to the manual](http://php.net/pcre).

#### Then the response body is an array of length `:length`

This step can be used to verify the exact length of a JSON array in the response body.

**Examples:**

*Assume that for the examples below, the response body is `[1, 2, 3]`.*

| Step                                             | :length | Test passes? |
| ------------------------------------------------ | ------- | ------------ |
| Then the response body is an array of length `1` | `1`     | No           |
| Then the response body is an array of length `3` | `3`     | Yes          |

If the response body does not contain a JSON array, an `InvalidArgumentException` exception will be thrown.

#### Then the response body should be an empty array

This is the same as `Then the response body is an array of length 0`.

#### Then the response body is an array with a length of at (`most`|`least`) `:length`

This step can be used to verify the length of an array, without having to be exact.

**Examples:**

*Assume that for the examples below, the response body is `[1, 2, 3, 4, 5]`.*

| Step                                                               | :length | Test passes? |
| ------------------------------------------------------------------ | ------- | ------------ |
| Then the response body is an array with a length of at `most` `4`  | `4`     | No           |
| Then the response body is an array with a length of at `least` `4` | `4`     | Yes          |
| Then the response body is an array with a length of at `most` `5`  | `5`     | Yes          |
| Then the response body is an array with a length of at `least` `5` | `5`     | Yes          |
| Then the response body is an array with a length of at `most` `6`  | `6`     | Yes          |
| Then the response body is an array with a length of at `least` `6` | `6`     | No           |

## Copyright / License

Copyright (c) 2016, Christer Edvartsen <cogo@starzinger.net>

Licensed under the MIT License
