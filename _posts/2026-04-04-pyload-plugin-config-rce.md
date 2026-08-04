---
layout: post
title: "CVE-2026-35463: the privilege boundary pyLoad forgot to extend to plugins"
summary: "An allowlist protected the dangerous core settings and stopped there. Plugin settings went through a different code path with no admin check at all, and one of them was an executable path."
reading_time: "6 min"
tags: [cve, command-injection, privilege-escalation, python]
lang: en
ref: pyload-plugin-config-rce
cve: CVE-2026-35463
---

**CVE-2026-35463** · [GHSA-w48f-wwwf-f5fr](https://github.com/advisories/GHSA-w48f-wwwf-f5fr) · pyLoad
(`pyload-ng`) · CWE-78 · High · published 2026-04-04

pyLoad is a self-hosted download manager. It has a role model: administrators, and users who
may hold a `SETTINGS` permission letting them change configuration without being admins.

That distinction is enforced by an allowlist:

```python
ADMIN_ONLY_OPTIONS = {
    "reconnect.script",   # a path that gets executed
    "webui.host",
    "ssl.cert_file",
    "ssl.key_file",
    # ...
}
```

The reasoning is sound. Some settings are not really settings, they are code execution with
extra steps, and those need to stay with the administrator.

## The gap

The check lives in the function that writes **core** configuration:

```python
def set_config_value(self, section, option, value):
    if f"{section}.{option}" in ADMIN_ONLY_OPTIONS:
        if not self.user.is_admin:
            raise PermissionError("Admin only")
```

Plugin configuration is written by a different path in the same file. That path has no check
at all:

```python
    self.pyload.config.set_plugin(category, option, value)
```

Two writers, one boundary. The allowlist only ever described core option names, so it could
never have matched a plugin option even if it had been consulted.

## The sink

pyLoad ships an `AntiVirus` add-on. It stores, in plugin configuration, the path of the
scanner binary to run and the arguments to pass it:

```python
def scan_file(self, file):
    avfile = self.config.get("avfile")
    avargs = self.config.get("avargs")
    subprocess.Popen([avfile, avargs, target])
```

`avfile` is an executable path, sitting in the config store that has no admin check, reachable
by any user holding `SETTINGS`. Point it at a binary of your choosing, trigger a scan, and
pyLoad executes it for you.

The interesting part is that this is precisely the risk `ADMIN_ONLY_OPTIONS` was written to
contain. `reconnect.script` is on the list because a settable script path is a settable
command. `avfile` is the same thing, one code path over.

## Why this shape keeps happening

This is a consistency gap, not an oversight in reasoning. Somebody understood the threat well
enough to build a defence and to enumerate the dangerous names. What did not happen is the
second question: *is this the only way a value reaches this store?*

Whenever you find a security control expressed as an allowlist or a denylist of names, that
question is worth asking on its own. The control is only as wide as the code path that
consults it, and configuration systems in mature projects almost always grow a second writer
eventually: plugins, profiles, environment overrides, an import feature.

I have found the same shape often enough that it is now one of the modelled patterns in my
research pipeline: locate a guard, then locate every writer that reaches the same store, and
diff the two sets.

## Impact and fix

A non-admin user with `SETTINGS` gets arbitrary command execution as the pyLoad process. In a
lot of deployments pyLoad runs as a service account with write access to the download tree,
which makes it a comfortable place to stage the next step.

The fix is to apply the same authorisation check to the plugin configuration writer, and to
treat any option whose value is a path or a command as admin-only regardless of which store
it lives in.

## Timeline

- Reported to the maintainers
- Advisory [GHSA-w48f-wwwf-f5fr](https://github.com/advisories/GHSA-w48f-wwwf-f5fr) published on 2026-04-04
- CVE-2026-35463 assigned
