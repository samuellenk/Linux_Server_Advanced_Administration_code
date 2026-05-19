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
ldapsearch -x -b "" -s base -LLL "+"
```

Bedeutung der Features:

- `supportedControl` = lässt Standardverhalten des Servers gegenüber Client ändern
- `supportedExtension` = zusätzliche Erweiterungen von Funktionen
- `supportedFeatures` = Server kann Client mitteilen, was unter unterstützt wird
- `supportedSASLMechanisms` = alle Verschlüsselungsmechanismen, die LDAP-Server für SASL unterstützt

# Einfache Filter

Suche nach allen Objekten mit dem Attribut `uid`:
```bash
ldapsearch -x -D "cn=admin,dc=example,dc=net" -b "dc=example,dc=net" -W \
  "(uid=*)"
```

UID mit `j` am Anfang:
```bash
ldapsearch -x -D "cn=admin,dc=example,dc=net" -b "dc=example,dc=net" -W \
  "(uid=j*)"
```

nur bestimmte Attribute anzeigen:
```bash
ldapsearch -x -D "cn=admin,dc=example,dc=net" -b "dc=example,dc=net" -W \
  "(uid=j*)" uid sn cn
```

# logischen Verknüpfungen

ODER-Verknüpfung:
```bash
ldapsearch -x -D "cn=admin,dc=example,dc=net" -b "dc=example,dc=net" -W \
  "(|(uid=j*)(uid=u*))" uid
```

UND-Verknüpfung:
```bash
ldapsearch -x -D "cn=admin,dc=example,dc=net" -b "dc=example,dc=net" -W \
  "(&(uid=j*)(loginShell=/bin/bash))" uid
```

Negation:
```bash
ldapsearch -x -D "cn=admin,dc=example,dc=net" -b "dc=example,dc=net" -W \
  "(&(!(uid=j*))(loginShell=/bin/bash))"
```

# Suchtiefe begrenzen

Suchtiefe begrenzen:
```bash
ldapsearch -x -D "cn=admin,dc=example,dc=net" -b "dc=example,dc=net" -W \
  -s sub "(&(!(uid=*n))(sn=*))"
```

Systemattribute zeigen:
```bash
ldapsearch -x -D "cn=admin,dc=example,dc=net" -b "dc=example,dc=net" -W \
  "(uid=s*)" +
```

# Einträge entfernen

User `tux` löschen:
```bash
ldapdelete -x -D "cn=admin,dc=example,dc=net" -W \
  -H ldap:/// "uid=john,ou=people,dc=example,dc=net"
```
