---
title: "Praxis-Beispiel: LDAP auf dem Client"
subtitle: "Linux Server Advanced Administration"
author: "![](../images/linux_trainings.png){width=20 height=20} [&copy; Linux-Trainings.de](https://linux-trainings.de/)"
theme: "Luebeck"
colortheme: "whale"
aspectratio: 169
colorlinks: true
urlcolor: gray
linkcolor: gray
---

# Einleitung

Diese Übung enthält ein vollständiges Beispiel zur Nutzung vom LDAP-Client

# Voraussetzung

- ein System (VM) mit Debian 13
- am **einfachsten das System** von der **LDAP-Server-Übung weiterverwenden**

# Zusammenfassung

Vor allem soll es um Suche in LDAP gehen, mithilfe der Befehle:

- `ldapsearch`
- `ldappasswd`
- `ldapadd`
- `ldapdelete`

# Fähigkeiten des LDAP-Servers testen

```bash
ldapsearch -x -b "" -s base + -LLL
```

Bedeutung der Features:

- `supportedControl` = lässt Standardverhalten des Servers gegenüber Client ändern
- `supportedExtension` = zusätzliche Erweiterungen von Funktionen
- `supportedFeatures` = Server kann Client mitteilen, was unter unterstützt wird
- `supportedSASLMechanisms` = alle Verschlüsselungsmechanismen, die LDAP-Server für SASL unterstützt

# Einfache Filter

Suche nach allen Objekten mit dem Attribut `uid`:
```bash
ldapsearch -x -D "uid=admin,dc=example,dc=net" -W "(uid=*)"
```

UID mit `s` am Anfang:
```bash
ldapsearch -x -D "uid=admin,ou=users,dc=example,dc=net" -W "(uid=s*)"
```

nur bestimmte Attribute anzeigen:
```bash
ldapsearch -x -D "uid=admin,ou=users,dc=example,dc=net" -W "(uid=s*)" cn sn
```

# logischen Verknüpfungen

ODER-Verknüpfung:
```bash
ldapsearch -x -D "uid=ldap-admin,ou=users,dc=example,dc=net" \
  -W "(|(uid=s*)(uid=u*))" uid
```

UND-Verknüpfung:
```bash
ldapsearch -x -D "uid=ldap-admin,ou=users,dc=example,dc=net" \
  -W "(&(uid=s*)(loginShell=/bin/bash))" uid
```

Negation:
```bash
ldapsearch -x -D "uid=ldap-admin,ou=users,dc=example,dc=net" \
  -W "(&(!(uid=c*))(loginShell=/bin/bash))"
```

# Suchtiefe begrenzen

Suchtiefe begrenzen:
```bash
ldapsearch -x -D "uid=ldap-admin,ou=users,dc=example,dc=net" \
  -W -b "ou=users,dc=example,dc=net" -s sub "(&(!(uid=*s))(sn=*))"
```

Systemattribute zeigen:
```bash
ldapsearch -x -D "uid=ldap-admin,ou=users,dc=example,dc=net" -W "(uid=s*)" +
```

# Einträge entfernen

User `tux` löschen:
```bash
sudo ldapdelete -x -D "cn=admin,dc=example,dc=net" \
  -W -H ldap://ldap.example.net \
  "uid=tux,ou=people,dc=example,dc=net"
```
