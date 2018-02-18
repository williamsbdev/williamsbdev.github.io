---
layout: post
title: "Ansible Lint testing"
tags: [ansible, testing]
---

As I have been building [Ansible] playbooks, I want to keep our Ansible
playbooks consistent and enforce proper formatting. There is a Python module
called [ansible-lint] that will enforce standardization of playbooks and
variable files. Most of the repos I build and use for work already have
automated Python unit tests and given I do not want to change the build script
to run the ansible-lint checks (or any other checks I'd like to add in the
future), I built a [test] with a [pytest] test fixure to gather up all the
files and run the linter on each file. Additionally, the test I built uses a
`pytest` test fixture, will build a test for each file and then with the
[pytest-xdist] module, I can run the tests in parallel to cutdown on the test
execution time. The [example] I provide, runs in 2.8 seconds (on my machine)
where if I run the example without the `pytest-xdist` module, it ran in 3.8
seconds. I only have 10 files in this repo but as it grows, the `pytest-xdist`
module will significantly cutdown on test execution time.

I've found that building a unit test to do checks like this, keeps our build
scripts simple as the only requirement is to run `pytest` instead of
customizing it for each new tool I would like to run a check over the codebase.
This is just a personal preference but I have found this method of adding
checks very valuable.

[Ansible]: https://www.ansible.com/
[ansible-lint]: https://github.com/willthames/ansible-lint
[test]: https://github.com/williamsbdev/ansible-examples/blob/master/tests/example_test.py
[pytest]: https://docs.pytest.org/en/latest/contents.html
[pytest-xdist]: https://pypi.python.org/pypi/pytest-xdist
[example]: https://github.com/williamsbdev/ansible-examples#run-tests-for-project
