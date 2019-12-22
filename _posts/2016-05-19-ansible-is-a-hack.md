---
title: Ansible is a hack on a hack on a hack on a hack
layout: post
category: tech
did-you-notice:
- This is YAML.
- It holds metadata related to this web page.
- It's human readable, and machines like it too.
- It's excellent for what it does.
- You didn't even know it was YAML until I showed you!
- {and: "here, a dict", look: [with, [nested], lists!]}
- Yet, did I abuse it?
---

Rant.

{% comment %}oh liquid...{% endcomment %}
{% raw %}

## Everything is JSON!

So the other day I wanted to pass a literal string, (that
coincidentally happened to contain valid JSON!), as a value of an
environment variable, because this is what the application needs.
Picture:

    env:
      SOME_CONFIG_OPTION: 1
      SOME_OTHER_OPTION: '{"a": 2}'

Can you imagine how much screwing around do you need to make this work
reliably in all contexts?

```djangotemplate
#!/bin/sh
{% for key, value in env.items() %}
export {{ key }}={{ value }}
{% endfor %}
exec ...
```

```yaml
tasks:
- action: foo
  environment: "{{ env }}"
```

Ansible will go way out of its way to make absolutely sure that the
contents of the string [will be interpreted as json][bug-15603],
turned into a python data structure, and then (most probably)
formatted using `repr`, or some equivalent, unless you try some
combination of dirty hacks:

- `{{ value | quote }}` won't work on dicts;
- `{{ value | to_json }}` doesn't properly escape for shell contexts;
- any combinations of `to_json` and `quote`, including some of:
  `to_json | to_json`, `to_json | quote`, `quote | to_json`, etc etc
  etc will all result in "*almost correct*" behavior; "*almost
  correct*" in this context translates to **utter garbage**;
- splitting `value` into two json-unparseable strings and cat'ing them
  together will still result in the final value being interpreted as
  literal json at some point;

So how to make this happen cleanly? After fighting for several hours,
I came up with the following:

- Create a custom filter plugin (!!!), e.g. add file
 `filter_plugsins/filters.py` in your Ansible repository:

    ```python
    import json

    def filter_to_envdict(x):
        """Quote values in dict for use as environment variable values.

        """
        env = {}
        if isinstance(x, dict):
            for key, value in x.iteritems():
                if isinstance(value, dict):
                    value = json.dumps(value)
                elif not isinstance(value, (str, unicode)):
                    value = repr(value)
                env[key] = value
        else:
            raise TypeError(type(x))
        return env

    class FilterModule(object):

        def filters(self):
            return {
                "to_envdict": filter_to_envdict,
            }

    ```

- Use as follows:

    ```djangotemplate
    #!/bin/sh
    {% for key, value in (env | default({}) | to_envdict).items() %}
    export {{ key }}={{ value | quote }}
    {% endfor %}
    exec ...
    ```

    ```yaml
    tasks:
    - action: foo
      environment: "{{ env | to_envdict }}"
    ```

Is this a solution? In that it has the end effect of making a
deployment possible - yes. As far as I'm concerned, it is a hack to
offset the effect of another hack.

This is just one tiny example, a symptom of a much bigger problem:
Ansible is built out of hacks.

[bug-15603]: https://github.com/ansible/ansible/issues/15603

## There is no clear boundary between languages

Ansible uses "vanilla" [YAML][] as the language describing its
playbooks, which means you can take any conforming YAML parser in the
world and just parse any of its playbooks.

[YAML]: http://yaml.org/

But there's a "problem" with YAML, that anyone runs into as soon as
they try doing anything fancy with it; it's not a programming
language! It's a configuration language, data serialisation and
exchange format; it's excellent when you don't feel like inventing a
configuration format for your tiny app, or when you need to embed
[metadata in a Markdown document](/tech/ansible-is-a-hack.md).

So Ansible has grafted [Jinja2][] on top of its YAML, which seems like
a great idea: Jinja is quite powerful, almost to the point of being a
"real" programming language in itself. (Let's disregard where did this
idea of [using a templating language as a programming language][PHP]
led us to in the recent past.)

[Jinja2]: http://jinja.pocoo.org/
[PHP]: https://www.php.net/

So what happens when you start freely mixing the two?

```yaml
- hosts: localhost
  become: no
  tasks:
  - action: {{ foo }}
```

```shell
% ansible-playbook foo.yml -e foo=ping
```

```
ERROR! Syntax Error while loading YAML.


The error appears to have been in '.../foo.yml': line 6, column 14, but may
be elsewhere in the file depending on the exact syntax problem.

The offending line appears to be:

  tasks:
  - action: {{ foo }}
             ^ here
We could be wrong, but this one looks like it might be an issue with
missing quotes.  Always quote template expression brackets when they
start a value. For instance:

    with_items:
      - {{ foo }}

Should be written as:

    with_items:
      - "{{ foo }}"

```

Oops. Ansible helpfully suggests we put quotes around our templated
string. Let's fix this quickly:

```yaml
- hosts: localhost
  become: no
  tasks:
  - action: "{{ foo }}"
```

```shell
% ansible-playbook foo.yml -e foo=ping
```

```
PLAY [localhost] ***************************************************************

TASK [setup] *******************************************************************
ok: [localhost]

TASK [{{ foo }}] ***************************************************************
ok: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0
```

Ansible has helpfully... ran the `ping` module, as asked on the
command line... but didn't bother to expand that same template when
printing the task name.

Until very recently (as of Ansible 1.9), this rule didn't apply
everywhere, and I'm sure even as 2.0 fixed a lot of things, there are
still spots where things break. If you'd try using template
substitution e.g. with an include statement:

```yaml
- hosts: localhost
  become: no
  tasks:
  - debug: var=ansible_distribution
  - include: "{{ ansible_distribution }}.yml"
```

Try!

```shell
% ansible-playbook -i /dev/null apply-foo.yml
```
```
ERROR: file could not read: .../{{ ansible_distribution }}.yml
```



Let's try another one: variables.

```yaml
- hosts: localhost
  become: no
  vars:
    a: "{{ b }}"
    b: ok
  tasks:
  - debug: var=a
```

Yes, we've seen this one before! It was in the programming 101 class.
This Python code will (quite obviously) fail on the first line:

```python
a = b
b = "ok"
print(a)
```

Then how come Ansible prints "ok"!? Go. Go check it. I'm not lying!

```shell
% ansible-playbook foo.yml
```

```
PLAY [localhost] ***************************************************************

TASK [setup] *******************************************************************
ok: [localhost]

TASK [debug] *******************************************************************
ok: [localhost] => {
    "a": "ok"
}

PLAY RECAP *********************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0
```

Note that the `vars` section is a dict, a hash table, an undordered
mapping - however you call it, there's no actual order to these pairs.

The only way this could ever happen is if there was some variable
resolution engine that tried templating all strings in various orders
until one way worked. What happens when we have more complex
dependencies? What happens when we have
[templates doing side-effects][PHP], like
[database lookups](https://docs.ansible.com/ansible/playbooks_lookups.html)?

I find it hard to believe the result can be deterministic.

## Is there a better way?

Once again, while
[Lisp failed to fix the world](https://www.quora.com/Where-did-we-go-wrong-Why-didnt-Common-Lisp-fix-the-world),
it had the correct answer from the very beginning.

Ansible, by nature, has to mix code and data - a lot; and achieves it
by patching a templating language into a data description language,
and then patching the data into the templates again - which constantly
falls apart at every seam.

Lisp is [homoiconic](https://en.wikipedia.org/wiki/Homoiconicity).
TLDR: this is a fancy word to say that code is data, and data is code.
It has templating built in right into its syntax. It has compile-time
macros, that are made in and of Lisp itself. It has data serialisation
built-in into the compiler. All of this is achievable in
[five thousand lines of quite clean and portable  C][tinyscheme].

[Common Lisp](https://common-lisp.net/) (in particular) is also
[pretty vast in scope][g10r]. But I'm not arguing for installing
Common Lisp on all your servers, that would be pretty insane, even if
perfectly in line with the current trend to install a
[whole horse](https://www.ruby-lang.org/)
[with legs](https://redis.io/) and
[a stable](https://www.erlang.org/), for example just to run your
[monitoring](https://sensuapp.org/).

[g10r]: https://en.wikipedia.org/wiki/Greenspun's_tenth_rule

What I'm arguing for, is to take inspiration from Lisp's design; after
all, we've successfuly incorporated [garbage][Java]
[collection][Csharp], [rich][Haskell] [typesystems][Fsharp],
[object][Java] [systems][Smalltalk], [exception][C++]
[handling][Java], [read][sh]-[eval][JS]-[print][Python] loops,
[object][Smalltalk] [introspection][Python] and [macros][Rust] into
several maintream languages and tools; and all of this is achievable
even in a [lightweight package][tinyscheme]. Why take a step back?

[C++]: https://en.wikipedia.org/wiki/C%2B%2B
[Csharp]: https://en.wikipedia.org/wiki/C_Sharp_%28programming_language%29
[Fsharp]: https://en.wikipedia.org/wiki/F_Sharp_%28programming_language%29
[Haskell]: https://en.wikipedia.org/wiki/Haskell
[JS]: https://en.wikipedia.org/wiki/JavaScript
[Java]: https://en.wikipedia.org/wiki/Java_%28programming_language%29
[Python]: https://en.wikipedia.org/wiki/Python_%28programming_language%29
[Rust]: https://en.wikipedia.org/wiki/Rust_(programming_language)
[Smalltalk]: https://en.wikipedia.org/wiki/Smalltalk
[sh]: https://en.wikipedia.org/wiki/Bourne_shell
[tinyscheme]: https://en.wikipedia.org/wiki/TinyScheme

## In Ansible's defense

With all of its warts and deficiencies, I couldn't imagine getting my
[current job][devops] done without Ansible - you can pry it from my
cold, dead hands!

[That is, until I get that suckless rewrite rolling.][judo]

[devops]: https://duckduckgo.com/?q=devops+%28buzzword%29
[judo]: https://github.com/rollcat/judo

{% endraw %}
