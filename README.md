# Puppet walkthrough

## Inhaltsverzeichnis

* [Best Practices für das Testen von Komponenten](#best-practices-für-das-testen-von-komponenten)
  * [Was ist ein Puppet Modul](#was-ist-ein-puppet-modul)
  * [Warum will man Linter?](#warum-will-man-linter)
  * [rspec](#rspec)
  * [rspec-puppet-facts](#rspec-puppet-facts)
  * [facterdb](#facterdb)
  * [beaker](#beaker)
  * [puppet-lint](#puppet-lint)
  * [puppet-syntax](#puppet-syntax)
  * [RuboCop](#rubocop)
  * [puppetlabs_spec_helper](#puppetlabs_spec_helper)
* [Wie veröffentlicht man erfolgreich Module und welchen Mehrwert bietet es?](#wie-veröffentlicht-man-erfolgreich-module-und-welchen-mehrwert-bietet-es)
  * [Wie strukturiert man in der Firma seinen Code sinnvoll?](#wie-strukturiert-man-in-der-firma-seinen-code-sinnvoll)
  * [Wie sieht ein gutes Komponentenmodul aus](#Wie-sieht-ein-gutes-komponentenmodul-aus)
  * [Mehrwert durch die Veröffentlichung von Komponentenmodulen](#Mehrwert-durch-die-veröffentlichung-von-komponentenmodulen)
  * [Mehrwert durch Communities](#mehrwert-durch-communities)
  * [modulesync](#modulesync)
  * [Weiterführende Dokumentation](#weiterführende-dokumentation)
* [Grundlagen für Continuous Integration](#grundlagen-für-continuous-integration)
  * [Grundlagen für interne CI](#grundlagen-für-interne-ci)
  * [Vorteile einer CI Pipeline](#vorteile-einer-ci-pipeline)
  * [Anforderungen an eine lokale CI/CD/CD Platform](#anforderungen-an-eine-lokale-cicdcd-platform)
  * [CI Dokumentation](#ci-dokumentation)
* [Grundlagen Continuous Delivery und Continuous Deployment](#grundlagen-continuous-delivery-und-continuous-deployment)
  * [Continuous Delivery](#continuous-delivery)
  * [Continuous Delivery für Puppet Module](#continuous-delivery-für-puppet-module)
  * [Continuous Deployment](#continuous-deployment)
* [Review der Puppetumgebung](#review-der-puppetumgebung)
  * [Grundsätzliches zum Tuning](#grundsätzliches-zum-tuning)
  * [PuppetDB](#puppetdb)
  * [PostgreSQL](#postgresql)
  * [Foreman](#foreman)
  * [Puppetserver](#puppetserver)
* [PDF](#pdf)
* [Lizenz](#lizenz)
* [Anmerkungen](#anmerkungen)

## Best Practices für das Testen von Komponenten

### Was ist ein Puppet Modul

* Ein normales Puppet Modul kann man auch als Komponentenmodul bezeichnen
* Es bündelt Puppet Klassen, welche eine einzelne Komponente verwalten (z.B. nginx oder Apache oder HAProxy)
  * Eine Klasse bündelt Puppet Ressourcen in einem Namespace. Dies hat wenig Klassen aus einem OOP Kontext zu sein
  * Man sollte eine Klasse in Puppet nicht mit einer Klasse in Java oder C# vergleichen
* Jedes Komponentenmodul sollte ein eigenes Git Repository haben
* Jedes Komponentenmodul ist nicht nur ein Puppet Projekt, sondern auch ein Ruby Projekt (weil `puppet` in Ruby geschrieben ist)
  * Ruby Projekte haben immer ein Gemfile, dies listet Ruby Abhängigkeiten für die Laufzeit, für Tests und für die Entwicklung
  * Ruby Projekte haben meistens ein Rakefile, dort sind Tasks definiert (ähnlich einem Makefile)
  * Ruby Projekte testet man (unter anderem) mit Ruby Tools

 Näheres dazu unter [puppetlabs_spec_helper](#puppetlabs_spec_helper).

### Warum will man Linter

* In Programmiersprachen, egal ob Puppet DSL, Ruby, Rspec DSL oder andere, gibt es viele Schreibweisen für das selbe Ergebniss
* Onboarding neuer Entwickler ist einfacher, wenn Code einheitlich ist
* Updates sind mit einheitlichem Code einfacher
  * Code auf eine neue Ruby Version portieren
  * Code auf eine neue Puppet Version portieren
* In jedem Ökosystem gibt es eine Community welche sich Style Guides ausdenkt, diesen kann man mit Lintern anwenden/erzwingen
* In einer [CI Pipeline]((#grundlagen-für-continuous-integration)) sind Linter gut, da sie sehr schnell arbeiten und direkt Feedback liefern
  * Optional auch als pre-commit Hook lokal ausführen
* Ein Linter prüft nicht zwingend den Code auf Syntax Fehler

### rspec

* rspec ist ein unit testing framework in Ruby. Es bringt eine eigene [DSL](https://en.wikipedia.org/wiki/Domain-specific_language) mit
* rspec-puppet ist eine Erweiterung für rpsec, um Puppet code testen zu können
* `Unit Tests` sind Tests, welche einzelne Funktionen / minimale Codeblöcke testen
* `rspec-puppet` validiert, ob bestimmte Puppet Resourcen so im Katalog sind, wie man sie erwartet
  * korrekte Parameter, Reihenfolge der Resourcen, Prüfen ob der Katalog kompiliert

### rspec Beispiele

Minimales Beispiel

```ruby
# import a helper
require 'spec_helper'

# class we want to test
describe 'borg' do
  # mock an FQDN
  let :node do
    'rspec.puppet.com'
  end

  let :facts do
    {
      "operatingsystem": "CentOS",
      "operatingsystemmajversion": 7
    }
  end

  # mock class params if required
  # bad practice, a module should work with default data
  let :params do
    {
      backupserver: 'localhost'
    }
  end

  context 'with all defaults' do
    it { is_expected.to compile.with_all_deps }
  end
end
```

* Gefühlt 80% der Probleme kann man damit lösen. Die meisten Fehler führen dazu, dass der Code nichtmal kompiliert

Prüfen der einzelnen Resources:

```ruby
require 'spec_helper'

describe 'borg' do
  let :node do
    'rspec.puppet.com'
  end

  on_supported_os.each do |os, facts|
    context "on #{os} " do
      let :facts do
        facts
      end

      let :params do
        {
          backupserver: 'localhost'
        }
      end

      context 'with all defaults' do
        it { is_expected.to compile.with_all_deps }

        it { is_expected.to contain_file('/etc/backup-sh-conf.sh') }
        it { is_expected.to contain_file('/etc/borg') }
        it { is_expected.to contain_file('/etc/profile.d/borg.sh') }
        it { is_expected.to contain_file('/usr/local/bin/borg-backup') }
        it { is_expected.to contain_file('/usr/local/bin/borg_exporter') }
        it { is_expected.to contain_file('/etc/borg-restore.cfg') }
        it { is_expected.to contain_class('borg::install') }
        it { is_expected.to contain_class('borg::config') }
        it { is_expected.to contain_class('borg::service') }
        it { is_expected.to contain_ssh__client__config__user('root') }
        it { is_expected.to contain_borg__ssh_keygen('root_borg') }
        it { is_expected.to contain_exec('ssh_keygen-root_borg') }
      end

      case facts[:os]['name']
      when 'Archlinux'
        context 'on Archlinux' do
          it { is_expected.to contain_package('borg') }
          it { is_expected.to contain_package('perl-app-borgrestore') }
        end
      when 'Ubuntu'
        context 'on Ubuntu' do
          it { is_expected.to contain_package('borgbackup') }
          it { is_expected.to contain_package('borgbackup-doc') }
          it { is_expected.to contain_package('gcc') }
          it { is_expected.to contain_package('make') }
          it { is_expected.to contain_package('cpanminus') }
          it { is_expected.to contain_package('libdbd-sqlite3-perl') }
          if facts[:os]['release']['major'] == '16.04'
            it { is_expected.to contain_apt__ppa('ppa:costamagnagianfranco/borgbackup') }
          end
        end
      when 'RedHat', 'CentOS'
        context 'on osfamily Redhat' do
          it { is_expected.to contain_package('perl-local-lib') }
          it { is_expected.to contain_package('perl-Test-Simple') }
          it { is_expected.to contain_package('perl-App-cpanminus') }
          it { is_expected.to contain_package('gcc') }
          it { is_expected.to contain_exec('install_borg_restore') }
          it { is_expected.to contain_file('/opt/BorgRestore') }
          it { is_expected.to contain_file('/usr/local/bin/borg-restore.pl') }
        end
      when 'Gentoo'
        context 'on osfamily Gentoo' do
          it { is_expected.to contain_package('App-cpanminus') }
        end
      when 'Fedora'
        context 'on osfamily Fedora' do
          it { is_expected.to contain_package('perl-Path-Tiny') }
          it { is_expected.to contain_package('perl-Test-MockObject') }
          it { is_expected.to contain_package('perl-Test') }
          it { is_expected.to contain_package('perl-autodie') }
        end
      end
    end
  end
end
```

Quelle ist das [puppet-borg](https://raw.githubusercontent.com/voxpupuli/puppet-borg/master/spec/classes/init_spec.rb) modul

* [rspec Homepage](https://rspec.info/)
* [rspec-puppet Homepage](https://rspec-puppet.com/)

## rspec-puppet-facts

* Puppet Module nutzen fast immer den `$facts` Hash
* In Tests muss man die Daten also mocken
* Das ganze sollte konsistent (z.B. `CentOS` vs `centos`) in allen Modulen sein
* Am liebsten automatisiert durch alle Betriebssysteme in der [metadata.json](https://github.com/voxpupuli/puppet-zabbix/blob/master/metadata.json#L62) iterieren und für jedes OS die Facts mocken

rspec-puppet-facts macht genau das!

```ruby
require 'spec_helper'

describe 'borg' do
  let :node do
    'rspec.puppet.com'
  end

  on_supported_os.each do |os, facts|
    context "on #{os} " do
      let :facts do
        facts
      end

      let :params do
        {
          backupserver: 'localhost'
        }
      end

      context 'with all defaults' do
        it { is_expected.to compile.with_all_deps }
      end
    end
  end
end
```

Es ist wichtig das alle Betriebssysteme, auf denen ein Modul genutzt wird, auch in der metadata.json stehen!

* [rspec-puppet-facts Homepage](https://github.com/mcanevet/rspec-puppet-facts#rspec-puppet-facts)

### facterdb

* rspec-puppet-facts ermöglicht das iterieren über die metadata.json in rspec tests
* die gemockten factsets kommen aus dme facterdb Projekt
* Sammlung an Scripten + Vagrant configs um VMs zu starten und darin facter Versionen zu installieren
* Gesammelten Daten werden als Ruby gem released

Für alle genutzten Betriebssysteme sollten in dem Projekt factsets sein!

* [FacterDB Homepage](https://github.com/camptocamp/facterdb#facterdb)
* [Facter documentation](https://puppet.com/docs/facter/3.11/index.html)
* Bugreport: [Facter 3.14 docs are broken](https://tickets.puppetlabs.com/browse/DOCUMENT-1076)

### Beaker

* Acceptance testing framework für Puppet
* Startet Virtualbox/Docker Instanz, führt darin das Puppet Modul aus
* Idempotenz kann getestet werden
  * Puppet Modul zwei mal ausführen, zweiter run darf keine Änderungen zeigen
* Mit rspec/serverspec kann das System inspiziert werden
  * Prüfen ob Packete installiert sind
  * Ob Services gestartet sind und laufen
  * Ob TCP ports offen sind
  * Und vieles mehr

```ruby
require 'spec_helper_acceptance'

describe 'zabbix::server class' do
  context 'default parameters' do
    # Using puppet_apply as a helper
    it 'works idempotently with no errors' do
      # this will actually deploy apache + postgres + zabbix-server + zabbix-web
      pp = <<-EOS
        class { 'postgresql::server': } ->
        class { 'zabbix::database': } ->
        class { 'zabbix::server': }
      EOS

      shell('yum clean metadata') if fact('os.family') == 'RedHat'

      # Run it twice and test for idempotency
      apply_manifest(pp, catch_failures: true)
      apply_manifest(pp, catch_changes: true)
    end

    # do some basic checks
    describe package('zabbix-server-pgsql') do
      it { is_expected.to be_installed }
    end

    describe service('zabbix-server') do
      it { is_expected.to be_running }
      it { is_expected.to be_enabled }
    end
  end
end
 ```

[Quelle der Tests](https://github.com/voxpupuli/puppet-zabbix/blob/master/spec/acceptance/server_spec.rb) ist das voxpupuli/zabbix Modul.

* Langfristig wird [OnyxPoint](https://www.onyxpoint.com/) die Entwicklung von beaker übernehmen
* Puppet Inc. arbeitet an Litmus, einer Ruby Bibliothek um VMs/Container zu starten
  * Tests sollen hier mit Bolt ausgeführt werden

* [Beaker Homepage](https://github.com/puppetlabs/beaker#beaker)
* [Serverspec Homepage](https://serverspec.org/)
* [Beaker-Puppet Homepage](https://github.com/puppetlabs/beaker-puppet#beaker-puppet-the-puppet-specific-beaker-library)
* [Beaker-Docker Homepage](https://github.com/puppetlabs/beaker-docker#beaker-docker)
* [Bolt Dokumentation](https://puppet.com/open-source/bolt/)

### puppet-lint

* Ruby Projekt um verschiedene Styles in der Puppet DSL zu checken / erzwingen
* Bietet ein Plugin-System
  * Jeder Check ist ein Plugin / eigenständiges Ruby gem
  * Die meisten Plugins haben eine Autokorrekturfunktion

Liste der aktuell empfohlenen plugins:

Gemfile:

```ruby
  gem 'puppet-lint-leading_zero-check',                             :require => false
  gem 'puppet-lint-trailing_comma-check',                           :require => false
  gem 'puppet-lint-version_comparison-check',                       :require => false
  gem 'puppet-lint-classes_and_types_beginning_with_digits-check',  :require => false
  gem 'puppet-lint-unquoted_string-check',                          :require => false
  gem 'puppet-lint-variable_contains_upcase',                       :require => false
  gem 'puppet-lint-absolute_classname-check', '>= 2.0.0',           :require => false
  gem 'puppet-lint-topscope-variable-check',                        :require => false
  gem 'puppet-lint-legacy_facts-check',                             :require => false
  gem 'puppet-lint-anchor-check',                                   :require => false
```

man kann in der Puppet DSL gezielt einzelne Linter deaktivieren, sofern sie ein False/Positive liefern:

```
# lint:ignore:case_without_default
```

Für Fehlermeldungen in einem sinnvollen Format kann man folgendes in seinem `Rakefile` eintragen:

```ruby
PuppetLint.configuration.log_format = '%{path}:%{line}:%{check}:%{KIND}:%{message}'
```

Für neue Komponentenmodule sollte man auch [puppet-lint-param-docs](https://github.com/voxpupuli/puppet-lint-param-docs#puppet-lint-parameter-documentation-check) nutzen. Dies erzwingt Dokumentation für jeden Parameter.


Ausgabe mit Fehler:

```
$ bundle exec rake lint
manifests/init.pp:106:arrow_on_right_operand_line:WARNING:arrow should be on the right operand's line
```

Ausgabe mit Autokorrektur:

```
$ bundle exec rake lint:auto_correct
manifests/init.pp:106:arrow_on_right_operand_line:FIXED:arrow should be on the right operand's line
```

* [puppet-lint Webseite](http://puppet-lint.com/)
* [Liste bekannter Community Plugins](http://puppet-lint.com/plugins/)
* [Liste der Vox Pupuli Plugins](https://github.com/voxpupuli?utf8=%E2%9C%93&q=puppet-lint&type=&language=)
* [Vox Pupuli hat weitere Style Guides, wofür es noch keine linter gibt](https://voxpupuli.org/docs/reviewing_pr/)

### puppet-syntax

* Checkt die Syntax der Puppet DSL mit `puppet parser`
* Checkt die Syntax von erb Templates mit Ruby
* Checkt die Syntax von EPP Templates mit `puppet`
* Checkt die Syntax von Hiera YAML Datein
  * Prüft auf valides YAML
  * Prüft auf korrekte Syntax der Keys, damit Puppet damit umgehen kann

Beispielhafte Ausgabe vom Rake Task wenn es keine Fehler gibt:

```
$ bundle exec rake syntax
  ---> syntax:manifests
  ---> syntax:templates
  ---> syntax:hiera:yaml
```

Ausgabe mit Syntaxfehlern:

```
$ bundle exec rake syntax
  ---> syntax:manifests
  rake aborted!
  Could not parse for environment production: Syntax error at end of file at demo.pp:2
  Tasks: TOP => syntax => syntax:manifests
  (See full trace by running task with --trace)
```

* [puppet-syntax Homepage](https://github.com/voxpupuli/puppet-syntax#puppetsyntax)
* [`puppet-parser` Dokumentation](https://puppet.com/docs/puppet/6.10/man/parser.html)
* [Dokumentation über ERB Templates in Puppet](https://puppet.com/docs/puppet/6.10/lang_template_erb.html)

### RuboCop

* RuboCop ist ein Linter für Ruby Datein
* Er lintet keinen Puppet DSL Code, aber eigenes Types und Provider
* Er lintet alle Datein mit Tests
* RuboCop ist sehr schmerzhaft...
  * Hat aber eine autofix Option die relativ gut funktioniert

Ausgabe mit Fehlern:

```
$ bundle exec rake rubocop
Running RuboCop...
Inspecting 4 files
..C.

Offenses:

spec/spec_helper.rb:8:1: C: Layout/EmptyLines: Extra blank line detected. (https://github.com/bbatsov/ruby-style-guide#two-or-more-empty-lines)

4 files inspected, 1 offense detected
RuboCop failed!
```

Mit Autokorrektur:

```
 bundle exec rake rubocop:auto_correct
Running RuboCop...
Inspecting 4 files
Parser::Source::Rewriter is deprecated.
Please update your code to use Parser::Source::TreeRewriter instead
..C.

Offenses:

spec/spec_helper.rb:8:1: C: [Corrected] Layout/EmptyLines: Extra blank line detected. (https://github.com/bbatsov/ruby-style-guide#two-or-more-empty-lines)

4 files inspected, 1 offense detected, 1 offense corrected
```

* [RuboCop Documentation](https://github.com/rubocop-hq/rubocop/blob/master/README.md)

### puppetlabs_spec_helper

* gem von Puppet Inc.
* Hat sehr viele der oben genannten Tools als Abhängigkeit
 * Puppet kümmert sich um korrekte Versionen die untereinander kompatibel sind. Man muss nur noch `puppetlabs_spec_helper` einbinden
* Viele Rake Tasks sind bereits mit dabei

Gemfile:
```ruby
source ENV['GEM_SOURCE'] || "https://rubygems.org"
gem 'puppetlabs_spec_helper'
```

Rakefile:

```ruby
require 'puppetlabs_spec_helper/rake_tasks'
```

Alle Rake Tasks ausgeben:

```sh
bundle exec rake -T
```

```
rake beaker                    # Run beaker acceptance tests
rake beaker:sets               # List available beaker nodesets
rake beaker:ssh[set,node]      # Try to use vagrant to login to the Beaker node
rake build                     # Build puppet module package
rake build:pdk                 # Build Puppet module with PDK
rake build:pmt                 # Build Puppet module package with PMT (Puppet < 6.0.0 only)
rake check:dot_underscore      # Fails if any ._ files are present in directory
rake check:git_ignore          # Fails if directories contain the files specified in .gitignore
rake check:symlinks            # Fails if symlinks are present in directory
rake check:test_file           # Fails if .pp files present in tests folder
rake clean                     # Clean a built module package
rake compute_dev_version       # Print development version of module
rake help                      # Display the list of available rake tasks
rake lint                      # Run puppet-lint
rake lint_fix                  # Run puppet-lint
rake parallel_spec             # Run spec tests in parallel and clean the fixtures directory if successful
rake parallel_spec_standalone  # Parallel spec tests
rake release_checks            # Runs all necessary checks on a module in preparation for a release
rake rubocop                   # rubocop is not available in this installation
rake spec                      # Run spec tests and clean the fixtures directory if successful
rake spec:simplecov            # Run spec tests with ruby simplecov code coverage
rake spec_clean                # Clean up the fixtures directory
rake spec_clean_symlinks       # Clean up any fixture symlinks
rake spec_list_json            # List spec tests in a JSON document
rake spec_prep                 # Create the fixtures directory
rake spec_standalone           # Run RSpec code examples
rake syntax                    # Syntax check Puppet manifests and templates
rake syntax:hiera              # Syntax check Hiera config files
rake syntax:manifests          # Syntax check Puppet manifests
rake syntax:templates          # Syntax check Puppet templates
rake validate                  # Check syntax of Ruby files and call :syntax and :metadata_lint
```

## Wie veröffentlicht man erfolgreich Module und welchen Mehrwert bietet es?

### Wie strukturiert man in der Firma seinen Code sinnvoll?

Anforderungen:

* Irgendwie muss man die ganzen [Komponentenmodule](#was-ist-ein-puppet-module-eigentlich) seinen Systemen zuweisen
* Man möchte öffentliche Komponentenmodule parallel zu selbst entwickelten nutzen
* Einige Systeme sind sehr ähnlich, aber doch anders
  * Jeder server hat SSH Passwort Authentifizierung deaktiviert, nur einer nicht
* Leute möchten Daten schreiben, aber sollen keinen Puppet Code modifizieren müssen/dürfen
  * Z.b. festlegen welche Version von interner Software wo läuft

Dies lässt sich mit dem Roles and Profiles Pattern lösen

* Ein Komponentenmodul kapselt Funktionalität um eine einzige Komponente zu verwalten
* Ein Profil enthält "Business-Logik"
  * Es kann die Reihenfolge von Komponentenmodulen setzen
    * z.B. epel Komponentenmodul vor Bird Komponentenmodul ausführen
  * Es holt sich Business Daten via Hiera
    * explizite `lookup()` Aufrufe oder [Automatic Class Parameter Lookup Pattern](https://puppet.com/docs/puppet/latest/hiera_automatic.html#class_parameters)
  * Mehrere solche Profile werden in einer Rolle gekapselt
  * Ein Profil kann in mehreren Rollen sein
  * Einem System wird immer nur eine Rolle zugewiesen
  * Profile und Rollen sind jeweils eigene Klassen

![roles-and-profiles-and-hiera](https://p.bastelfreak.de/daRFU/)

(Copyright Craig Dunn)

### Wie sieht ein gutes Komponentenmodul aus

* `include modul` funktioniert und installiert mir eine Komponente mit sinnvollen und sicheren Standardwerten
* Es gibt eine README.md
* Es gibt eine Lizenz
* Es enthält keine Business-Logik
  * Irgendetwas, was firmenspezifisch ist
  * Das Modul sollte in jeder Umgebung funktionieren
  * Ansonsten wird das [Roles and Profiles Pattern]((#wie-strukturiert-man-in-der-firma-seinen-code-sinnvoll)) nicht eingehalten
* Es gibt Unit- und Acceptance Tests
* Aktuelle [Style Guidelines](#warum-will-man-linter) werden eingehalten
* Parameter mit `puppet-strings` dokumentiert

### Mehrwert durch die Veröffentlichung von Komponentenmodulen

* Ein internes Modul nachträglich veröffentlichen kann Security Probleme mitbringen
  * Es kann potentiell Business-Daten enthalten (Passwörter, IP-Adressen...)
* Ist an Modul von Anfang an öffentlich:
  * achtet man auf eine sichere Implementierung
  * Enthält es nie Business-Logik. Diese wird dann im Profil implementiert. Dadurch is das Modul wiederverwendbar
* Sofern Dokumentation vorhanden ist, wird das Modul von anderen benutzt
  * Man bekommt plötzlich Bug Reports und Feature Requests, die potentiell mehr / unvorgergesehene Arbeit erzeugen
  * Man bekommt auch Pull Requests mit neuen Features und auch Bug Fixes

Es kommt vor, dass man ein öffentliches Modul patchen muss. Man muss versuchen
diese Patches Upstream gemerged zu bekommen. Andernfalls difft der Fork immer
mehr von Upstream ab und man landet in der Abhängigkeitshölle. Bei Modulen von
Vox Pupuli / Puppet wird tendenziell schneller reagiert als bei Modulen mit
einzelnen Maintainern. Puppet hat viele Mitarbeiter mit Commit Rechten für Ihre
Module, Vox Pupuli hat > 140 (unregelmäßige) Maintainer. Außerdem gibt es hier
Esakalationsmöglichkeiten.

### Mehrwert durch Communities

* Vox Pupuli, Foreman, CERN, CamptoCamp, Puppet Inc und weitere Gruppen / Organisationen / Firmen pflegen viele Komponentenmodule
* Ein Team kümmert sich um alle generischen Datein in den Modulen
  * Einheitliche Testmatrix
  * Einheitliches Testsetup
  * Einheitliches Gemfile
* Die ganze Community hilft beim beantworten von Issues / Pull Requests
* Oftmals findet man hier schon ein fertiges Modul für seinen Usecase

### modulesync

* modulesync wurde von Puppet Angestellten entwickelt
* Von Anfang an Open Source, mittlerweile von Vox Pupuli betreut
* In einem Git Repository liegen Templates, diese werden auf alle verwalteten Module angewand
* Änderungen kann man auf Modul Ebene überschreiben
* Eine Hand voll Leute können damit >120 Module verwalten
* Modulesync definiert auch die komplette Testmatrix für alle Module
* Viele Firmen nutzen die Vox Pupuli Konfigurationen für interne Module / man kann modulesync aber auch unabhängig benutzen

### Weiterführende Dokumentation

* [Erfindung des Roles and Profiles Pattern (Craig Dunn)](https://www.craigdunn.org/2012/05/239/)
* [Deepdive von Gary Larizza](http://garylarizza.com/blog/2014/02/17/puppet-workflow-part-2/)
* [Roles and Profiles Dokumentation (Puppet Inc.)](https://puppet.com/docs/pe/2019.2/designing_system_configs_roles_and_profiles.html)
* [puppet-strings Dokumentation](https://puppet.com/docs/puppet/latest/puppet_strings.html)
* [Vox Pupuli Module auf forge.puppet.com](https://forge.puppet.com/puppet?page_size=25&sort=downloads)
* [Modulesync Dokumentation](https://github.com/voxpupuli/modulesync#modulesync)
* [Modulesync Konfiguration für Vox Pupuli](https://github.com/voxpupuli/modulesync_config#modulesync-configs)

## Grundlagen für Continuous Integration

* Jede Änderung sollte getestet werden
* Testergebnisse sollten möglichst schnell sichbar sein
* Tests müssen reproduzierbar sein
* Tests müssen in einer sauberen Umgebung ausgeführt werden, z.B. nspawn/Docker Container oder einer VM
* Tests müssen auf einer zentralen Platform ausgeführt werden
  * Jenkins
  * Gitlab-CI
  * Travis
* Je mehr öffentliche Module man nutzt, desto weniger muss man lokal testen
  * Die einzelnen Communities haben eine eigene Testinfrastruktur mit passender testmatrix
* Acceptance Tests für Controlrepos sind komplex zu bauen. [Onceover](https://github.com/dylanratcliffe/onceover#onceover) hilft dabei
  * Ordentliche Tests in allen Komponentenmodulen ist wesentlich einfacher
  * Acceptance Tests in jedem Komponentenmodul sollte Daten aus den eigenen Profilen enthalten
* Prüft jede metadata.json:
  * Alle benötigten Module müssen als Dependencies gelistet sein
  * Alle Betriebssysteme müssen dort gelistet sein
* Spec Tests müssen rspec-puppet-facts nutzen
* Alle genutzten Betriebsysteme müssen in facterdb sein
* Irgendwo muss noch ein Peer Review vorkommen

### Grundlagen für interne CI

* Man benötigt eine Testmatrix. Für Unit Tests (im Optimalfall):
  * Gegen alle Ruby Versionen die man nutzt
  * Gegen alle Puppet Versionen die man nutzt
  * Gegen alle Betriebssysteme die man nutzt
* Automatische Lizenzprüfung aller Abhängigkeiten
* CPU Leistung: Viel hilft viel!
  * Wenn genug Leistung zur Verfügung steht, kann man für jeden Test eine Dockerinstanz parallel starten
* Abhängigkeiten: Fehler früh erkennen
  * In der Regel nutzt man für Unit Tests die Master Branches der Abhängigkeiten, um noch nicht releaste Inkompatibilitäten früh zu erkennen
  * Für Acceptance Tests nutzt man die neusten Releases der Abhängigkeiten
* Bei Neuen Modulversionen:
  * Die Abhängigkeiten in der ganzen Environment prüfen mit `puppet module list --environment production --tree`
  * In `/var/log/puppetlabs/puppetserver/puppetserver.log` nach Warnungen / Fehlern schauen

### Vorteile einer CI Pipeline

* Steigerung der Codequalität
* Erzwingen einer möglichst einheitlichen Codebasis
* Potentielle Fehler/Bugs/unerwünschtes Verhalten erkennen bevor es passiert
* Inkompatibilitäten in der Zukunft erkennen und vorbeugen, damit diese nicht eintreten
* Gibt Mitarbeitern mehr Selbstbewusstsein Änderungen zu machen
  * Verringert auch die Einstiegshürde für andere Administratoren

### Anforderungen an eine lokale CI/CD/CD Platform

* Ergebnisse müssen schnell und einfach ersichtlich sein
* Testumgebungen müssen schnell verfügbar sein
  * R10k nutzen zum ausrollen von Environments
  * tar.gz Artefakte oder git tags/commits deployen, keine branches
  * Git Server Webhooks nehmen und Testenvironments zu erstellen sobald Commits / Branches gepusht werden
    * Puppet Webhook Server + Choria / mcollective / Bolt zum ausrolen

SSH Multiplexing aktivieren um `git clone`s zu beschleunigen:

 ```puppet
 class profiles::ssh2git {
  # /root/.ssh/config entry to access git.internal
  ssh::client::config::user { 'gitaccess':
    ensure        => present,
    user          => 'root',
    user_home_dir => '/root',
    options       => {
      'Host git.internal' => {
        'User'           => 'git',
        'IdentityFile'   => '~/.ssh/id_ed25519...',
        'ControlMaster'  => 'auto',
        'ControlPath'    => '~/.ssh/ssh-%r@%h:%p',
        'ControlPersist' => '10m'
      },
    },
  }
  sshkey { 'git.internal':
    ensure       => 'present',
    target       => '/root/.ssh/known_hosts',
    type         => 'ecdsa-sha2-nistp256',
    key          => 'DATA',
  }
}
 ```

r10k mit mehreren Workern starten:

```puppet
# puppet/r10k from Vox Pupuli
class { 'r10k':
  version   => '3.5.0',
  pool_size => 10,
}
```

### CI Dokumentation

* [Puppet Catalog Diff Viewer von CamptoCamp](https://github.com/camptocamp/puppet-catalog-diff-viewer#puppet-catalog-diff-viewer)
* [octocatalog-diff Homepage](https://github.com/github/octocatalog-diff#octocatalog-diff-)
* [Installation von octocatalog-dif (Example42)](https://www.example42.com/2018/03/05/catalog-diff-on-refactoring/)
* [Onceover Dokumentation](https://github.com/dylanratcliffe/onceover#onceover)
* [Puppet Webhook Server Dokumentation](https://github.com/voxpupuli/puppet_webhook#puppet-webhook-server)
* [Choria Webseite](https://choria.io/)
* [Vortrag von Kevin Paulisse (GitHub Inc.) über CI](https://www.youtube.com/watch?v=TkzrUT-ZPQ8)

## Grundlagen Continuous Delivery und Continuous Deployment

### Continuous Delivery

* Continuous Delivery - nach einem erfolgreichen CI Durchlauf Buildartefakte bauen und in einem Repository speichern
* Artefakte können verschiedenste Formen haben
  * Ruby gem / Python Package
  * Docker Image
  * rpm / deb Paket
  * Puppet Modul
  * Ein generisches .tar.gz Archiv

### Continuous Delivery für Puppet Module

Es gibt viele Möglichkeiten Puppet Module zu speichern.

Folgende Projekte unterstützen Puppet Modul Releases(.tar.gz Archive):

* Artifactory
* Pulp
* Sonatype Nexus
* [kindredgroup/puppet-forge-server)](https://github.com/kindredgroup/puppet-forge-server)

Alternativ kann man mit git tags / commit ids im Puppetfile arbeiten.

### Continuous Deployment

* Continuous Deployment - Build Artefakte möglichst automatisiert in die Produktivumgebung bringen

Mit Puppet eine große Menge an Scripten als File Resources verteilen ist ein
Antipattern. Diese sollte man als Artefakte Paketieren. Puppet richtet dann das
Repository auf den Zielsystemen ein. Dies gillt für alle Artefakte, welche
Puppet deployen soll.

Bei vielen Projekten bietet sich ein [gitflow-workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow) an:

* Develop Umgebung läuft auf einem individuellen Dev Branch
* Test Umgebung läuft auf dem Develop Branch
* Staging/Integreation Umgebung läuft auf Master Branch
* Git Tags basieren auf Master Branch. Diese werden nach Produktion deployed

Dies funktioniert sehr gut für intern entwickelte Anwendungen die auf anderen
internen Tools aufsetzen und wiederum von anderen Teams genutzt werden. Für
jede Umgebung benötigt man eigene Repositories und Maschinen/Server/Docker
Images. Puppet kann solche Umgebungen deployen. Puppet Code selbst mit so einem
Workflow zu verwalten funktioniert in der Regel nicht / bringt viel zu viel
Overhead mit sich.

Für Puppet hat sich in den meisten Organisationen ein [Git Feature Branch Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/feature-branch-workflow) als beste Lösung etabliert.

* Im Control-Repository ist der Standardbranch `production`. Dieser wird als Environment auf normalen Nodes genutzt
* Zum testen / entwickeln erzeugt man einen Feature Branch im Control-Repository + ggf im entsprechenden Modul
* Push eines Branches im Control-Repo erzeugt via hooks die Environment auf den Puppetservern
* Bei Merges wird Production neu deployed

## Review der Puppetumgebung

Dedizierter Vortrag über  [Puppetserver Skalierung](https://github.com/bastelfreak/scalingpuppetserver#scaling-puppetserver)

### Grundsätzliches zum Tuning

* PuppetDB / Puppetserver CA / Compile Puppetserver / Foreman / PostgreSQL / Choria haben verschiedene Anforderungen an Hardware und kann man gut auf einzelnen Systemen installieren
* Ein verteiltes System ist immer nur so stark wie ihr schwächstes Glied
* Tuning Optionen sind immer stark abhängig von der Anzahl der Resources pro Node / Runinterval / Funktionen
  * Die Werte sollten deshalb nicht blind übernommen werden

### PuppetDB

Hiera Tuningoptionen für [puppetlabs/puppetdb](https://forge.puppet.com/puppetlabs/puppetdb):

```yaml
---
puppetdb::server::java_args:
  '-Xmx': '8192m'
  '-Xms': '2048m'
puppetdb::server::node_ttl: '14d',
puppetdb::server::node_purge_ttl: '14d',
puppetdb::server::report_ttl: '999d'
# default is 50
puppetdb::server::max_threads: 100
# default is processorcount / 2
puppetdb::server::command_threads: %{facts.processors.count}
# default is 4, have your database in mind
puppetdb::server::concurrent_writes: 8
puppetdb::server::automatic_dlo_cleanup: true
```

### PostgreSQL

Optionen für [puppetlabs/postgresql](https://forge.puppet.com/puppetlabs/postgresql):

Anzahl der Verbindungen erhöhen:

```puppet
postgresql::server::config_entry{'max_connections':
  value => 400,
}
```

Laufenden PostgreSQL Server analysieren und eine optimierte Konfiguration generieren:

```sh
pgtune -i /var/lib/pgsql/10/data/postgresql.conf -o postgresql.conf
```

Für PostgreSQL empfiehlt sich deren eigenes Yum Repository. Die Upstream Pakete
erlauben es beliebige Versionen parallel zu installieren. Außerdem bekommt das
Yum Repository am schnellsten Sicherheitsupdates.

### Foreman

Hiera Optionen für [theforeman/foreman](https://forge.puppet.com/theforeman/foreman):

```yaml
---
# default is 5
foreman::db_pool: 20
foreman::keepalive: true
foreman::max_keepalive_requests: 1000
foreman::keepalive_timeout: 180
```

Foreman läuft via Passenger. Dieser wird mit
[puppetlabs/apache](https://forge.puppet.com/puppetlabs/apache) aufgesetzt.
Multithreading hochdrehen:

```yaml
---
apache::mod::passenger::passenger_max_pool_size: %{facts.processors.count}
apache::mod::passenger::passenger_min_instances: %{facts.processors.count}
```

Foreman kann memcached als Cache nutzen:

Via Puppet installieren:

```puppet
# https://forge.puppet.com/saz/memcached
include memcached
include foreman::plugin::memcache
```

Via Hiera einstellen:

```yaml
---
# 50GB of cache
memcached::max_memory: 51200
foreman::plugin::memcache::hosts:
  - 127.0.0.1
```

### Puppetserver

Puppetlabs bietet im Modul
[puppetlabs/puppet_metrics_dashboard](https://forge.puppet.com/puppetlabs/puppet_metrics_dashboard)
einen Monitoring Stack für Puppetserver. Dieser bietet JVM Metriken via JMX und
Puppetserver Metriken via Graphite.

* Eine JVM Instanz pro CPU Kern funktioniert
* ~800-1500 Resources pro katalog vertragen 2GB pro JVM Instanz
* Puppetserver kann zum starten weniger Ram allokieren, dies verkürzt die Bootstrap Zeit
* Puppetserver hat einen Class Cache, welchen man aktivieren kann
* Jede JVM Instanz hat einen Cache, dieser wird besser je länger er läuft
  * Serverseitige Funktionen mit memory Leaks führen zu Problemen
  * Jeder Restart tötet den Cache
  * server_max_requests_per_instance auf unendlich setzen oder mindestens 100.000
* Mehrere Puppetserver kann man via DNS Round Robin skalieren
* HAProxy/Nginx als Proxy davor funktioniert besser als Round Robin
* nginx wurde gern zum terminieren von TLS Verbindungen direkt auf Puppetservern genutzt
  * Puppetserver kann dies seit 6 selbst gut
  * Legacy Vortrag: [bastelfreak.de/scalingpuppetserver](https://bastelfreak.de/scalingpuppetserver) ([Repo](https://github.com/bastelfreak/scalingpuppetserver))
* `file` Reports sind unnötig und fressen oft IO/s und Inodes (`grep ^reports /etc/puppetlabs/puppet/puppet.conf`)

Hiera Optionen für [theforeman/puppet](https://forge.puppet.com/theforeman/puppet):

```puppet
$cpu_count_twice = $facts['processors']['count'] * 2
$cpu_count = $facts['processors']['count'] * 1
class{'puppet':
  server_jvm_min_heap_size               => "${cpu_count}G",
  server_jvm_max_heap_size               => "${cpu_count_twice}G",
  server_max_requests_per_instance       => 100000,
  server_max_queued_requests             => $cpu_count,
  server_environment_class_cache_enabled => true,
  server_jvm_extra_args                  => ['-XX:ReservedCodeCacheSize=2G'],
}
```

## PDF

`pandoc` ermöglicht es aus dieser markdown Datei eine pdf zu generieren. Die
gängigen Linux Distributionen haben pandoc in ihren Repositories. Der simpelste
Aufruf lautet wie folgt:

```sh
pandoc --from markdown README.md -o puppet.pdf
```

```sh
pandoc README.md -o puppet.pdf "-fmarkdown-implicit_figures -o" --from=markdown -V geometry:margin=.4in --toc --highlight-style=espress
```

## Lizenz

Dieses Dokument steht unter der
[CC BY-NC-SA 4.0 Lizenz](https://creativecommons.org/licenses/by-nc-sa/4.0/).
Codebeispiele sind unter der [GNU Affero General Public License v3.0](LICENSE)
lizensiert. Für kommerzielle Anfragen, Nachfragen zu Consulting oder Feedback
kontaktieren sie bitte [tim@bastelfreak.de](mailto:tim@bastelfreak.de).

## Anmerkungen

* Dieses Dokument wurde ursprünglich als gist gepflegt: [https://gist.github.com/bastelfreak/33a9d1510448e31e7d8139a80d16e44a](https://gist.github.com/bastelfreak/33a9d1510448e31e7d8139a80d16e44a)
* Weitere Vorträge und Dokumente gibt es in meinem [Git Repository](https://github.com/bastelfreak/talks#collection-of-talks-proposals-and-related-stuff)
