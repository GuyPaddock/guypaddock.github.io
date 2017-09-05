---
layout: article
title: "Decoding the Number in JSON::ParserError messages"
categories: posts
modified: 2017-09-05T12:00:00-04:00
tags: [ruby, json, ruby on rails]
comments: true
ads: false
---

This is just a quick post to share something I just learned about the `json` gem. Recently, we upgraded from version 1.8.3 to 1.8.6 of `json`, and observed that one of our tests that ensure we gracefully handle bad JSON from one of our vendors started to fail.

Our RSpec test was expecting this:
```
Rescued MapQuest API `JSON::ParserError`. Error:
795: unexpected token at 'You have exceeded the number of monthly transactions included with your current plan. If you need additional transactions, please consider upgrading to a plan that offers additional transactions. If you would like to talk to an Account Manager about Enterprise Edition licensing options, please contact sales@mapquest.com.'
```

But we started to get this:
```
Rescued MapQuest API `JSON::ParserError`. Error:
822: unexpected token at 'You have exceeded the number of monthly transactions included with your current plan. If you need additional transactions, please consider upgrading to a plan that offers additional transactions. If you would like to talk to an Account Manager about Enterprise Edition licensing options, please contact sales@mapquest.com.'
```

Notice the difference? The only difference was that after the upgrade we were now getting 822 instead of 795 at the start of the message. But... what does 795 / 822 actually represent? An error code? The length of the response? The line number in the source JSON? The line number in Ruby? There didn't seem to be any obvious answer -- especially because the error is the same length and much shorter than 822 characters.

I also checked the stack for the test-induced exception, to no avail:
```
/home/vagrant/.rvm/gems/ruby-2.0.0-p643/gems/json-1.8.6/lib/json/common.rb:155:in `parse'
/home/vagrant/.rvm/gems/ruby-2.0.0-p643/gems/json-1.8.6/lib/json/common.rb:155:in `parse'
/home/vagrant/.rvm/gems/ruby-2.0.0-p643/gems/httparty-0.13.7/lib/httparty/parser.rb:118:in `json'
/home/vagrant/.rvm/gems/ruby-2.0.0-p643/gems/httparty-0.13.7/lib/httparty/parser.rb:138:in `parse_supported_format'
/home/vagrant/.rvm/gems/ruby-2.0.0-p643/gems/httparty-0.13.7/lib/httparty/parser.rb:105:in `parse'
/home/vagrant/.rvm/gems/ruby-2.0.0-p643/gems/httparty-0.13.7/lib/httparty/parser.rb:67:in `call'
/home/vagrant/.rvm/gems/ruby-2.0.0-p643/gems/httparty-0.13.7/lib/httparty/request.rb:322:in `parse_response'
/home/vagrant/.rvm/gems/ruby-2.0.0-p643/gems/httparty-0.13.7/lib/httparty/request.rb:295:in `block in handle_response'
/home/vagrant/.rvm/gems/ruby-2.0.0-p643/gems/httparty-0.13.7/lib/httparty/response.rb:23:in `call'
/home/vagrant/.rvm/gems/ruby-2.0.0-p643/gems/httparty-0.13.7/lib/httparty/response.rb:23:in `parsed_response'
/home/vagrant/.rvm/gems/ruby-2.0.0-p643/gems/httparty-0.13.7/lib/httparty/response.rb:66:in `method_missing'
/home/vagrant/my_project/lib/map_quest_client.rb:31:in `optimized_route'
```

Certainly, no 822 showing up there.

The [docs for JSON::ParserError](https://ruby-doc.org/stdlib-1.9.3/libdoc/json/rdoc/JSON/ParserError.html) were similarly useless. Beyond a one-sentence description, no detail on what numbers in the message, if any, should mean.

As it turns out, the answer lies in C code within the [JSON gem](https://github.com/flori/json). I checked out the project at tag `1.8.6` and started my search there. _(As an aside: WTF happened with this gem that they needed to skip version 1.8.4? They went from 1.8.3 to 1.8.5 and all the release notes say is that they skipped it, but... WHY?)._

Grepping through the project using `grep -R ": unexpected token at" -n .` I got:
```
./ext/json/ext/parser/parser.c:553:                rb_enc_raise(EXC_ENCODING eParserError, "%u: unexpected token at '%s'", __LINE__, p);
./ext/json/ext/parser/parser.c:589:            rb_enc_raise(EXC_ENCODING eParserError, "%u: unexpected token at '%s'", __LINE__, p - 8);
./ext/json/ext/parser/parser.c:599:            rb_enc_raise(EXC_ENCODING eParserError, "%u: unexpected token at '%s'", __LINE__, p - 2);
./ext/json/ext/parser/parser.c:1319:        rb_enc_raise(EXC_ENCODING eParserError, "%u: unexpected token at '%s'", __LINE__, p);
./ext/json/ext/parser/parser.c:1926:        rb_enc_raise(EXC_ENCODING eParserError, "%u: unexpected token at '%s'", __LINE__, p);
./ext/json/ext/parser/parser.c:2099:        rb_enc_raise(EXC_ENCODING eParserError, "%u: unexpected token at '%s'", __LINE__, p);
./ext/json/ext/parser/parser.rl:229:            rb_enc_raise(EXC_ENCODING eParserError, "%u: unexpected token at '%s'", __LINE__, p - 2);
./ext/json/ext/parser/parser.rl:236:            rb_enc_raise(EXC_ENCODING eParserError, "%u: unexpected token at '%s'", __LINE__, p - 8);
./ext/json/ext/parser/parser.rl:252:                rb_enc_raise(EXC_ENCODING eParserError, "%u: unexpected token at '%s'", __LINE__, p);
./ext/json/ext/parser/parser.rl:419:        rb_enc_raise(EXC_ENCODING eParserError, "%u: unexpected token at '%s'", __LINE__, p);
./ext/json/ext/parser/parser.rl:784:        rb_enc_raise(EXC_ENCODING eParserError, "%u: unexpected token at '%s'", __LINE__, p);
./ext/json/ext/parser/parser.rl:822:        rb_enc_raise(EXC_ENCODING eParserError, "%u: unexpected token at '%s'", __LINE__, p);
```

There it is! It turns out that the number in the exception represents a line number in the `.rl` file that is pre-processed by [Ragel(http://www.colm.net/open-source/ragel/) to generate the actual JSON parser in C. The number indicates which line in the parser had an issue with the code!

To confirm, if I roll back the gem to 1.8.3 and run the same query, I get:
```
./ext/json/ext/parser/parser.c:533:                rb_raise(eParserError, "%u: unexpected token at '%s'", __LINE__, p);
./ext/json/ext/parser/parser.c:569:            rb_raise(eParserError, "%u: unexpected token at '%s'", __LINE__, p - 8);
./ext/json/ext/parser/parser.c:579:            rb_raise(eParserError, "%u: unexpected token at '%s'", __LINE__, p - 2);
./ext/json/ext/parser/parser.c:1299:        rb_raise(eParserError, "%u: unexpected token at '%s'", __LINE__, p);
./ext/json/ext/parser/parser.c:1899:        rb_raise(eParserError, "%u: unexpected token at '%s'", __LINE__, p);
./ext/json/ext/parser/parser.c:2072:        rb_raise(eParserError, "%u: unexpected token at '%s'", __LINE__, p);
./ext/json/ext/parser/parser.rl:209:            rb_raise(eParserError, "%u: unexpected token at '%s'", __LINE__, p - 2);
./ext/json/ext/parser/parser.rl:216:            rb_raise(eParserError, "%u: unexpected token at '%s'", __LINE__, p - 8);
./ext/json/ext/parser/parser.rl:232:                rb_raise(eParserError, "%u: unexpected token at '%s'", __LINE__, p);
./ext/json/ext/parser/parser.rl:399:        rb_raise(eParserError, "%u: unexpected token at '%s'", __LINE__, p);
./ext/json/ext/parser/parser.rl:757:        rb_raise(eParserError, "%u: unexpected token at '%s'", __LINE__, p);
./ext/json/ext/parser/parser.rl:795:        rb_raise(eParserError, "%u: unexpected token at '%s'", __LINE__, p);
```

And, there we have it -- `795` -- just like we had in our original exception message. So, it's a line number... in Ragel code.
![The More You Know](https://giphy.com/gifs/star-shooting-the-more-you-know-3og0IMJcSI8p6hYQXS)
