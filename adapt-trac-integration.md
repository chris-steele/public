# Adapt/Trac integration

The `adapt-trac` plugin provides an Adapt module with an integrated [Trac](https://trac.edgewall.org/) user interface; allowing multiple reviewers to create and review tickets in situ.

The plugin can be used both in standalone [Adapt Framework](https://github.com/adaptlearning/adapt_framework) builds and [Adapt Authoring Tool](https://github.com/adaptlearning/adapt_authoring) (AAT) instances.

## Configuration

The following instructions describe how to configure both the Adapt module and the Trac instance to use the plugin. Further instructions cover deployment which falls into two categories: deployment from a client site (`clients.kineo.com/[client]`) or from other platforms such as the subversion server (`dev.kineo.com`) or an AAT instance.

Client sites use a Moodle plugin that has been developed to support the Adapt/Trac integration. When you are ready to deploy an Adapt module, open the appropriate client site and create a new course if one is not present (`Site Administration > Courses > Add/edit courses`). To avoid ambiguity _course_ shall refer to the location on Moodle where Adapt content is placed. Reference to each individual piece of Adapt content shall continue to use _module_. Keep in mind that there may be multiple Adapt modules in a course (e.g. `m05`, `m10` etc.).

## 2.1 Adapt configuration 

Some small changes are required in the Adapt configuration (`course.json` and `config.json`). Note that these changes are required for each Adapt module. 

### 2.1.1 course.json 

In `course.json` create a `_trac` configuration by using the exemplar found in `example.json`. Alternatively, if you are using the AAT, install and enable the `adapt-trac` plugin for the module and then go to the `Trac` extension settings within `Project settings`.

Ensure that the `_id` property corresponds to a Trac component name (see [2.2.2 Add Components](#222-add-components)). For example; module should have an `_id` of `m05` and there should be a component named `m05` configured in Trac.

### 2.1.2 config.json 

In `config.json` locate the `_scrollingContainer` property group and set `_isEnabled` to `true` and `_limitToSelector` to `.no-touch`. Alternatively, if you are using the AAT, locate the `iFrame and Screen Reader scrolling support` settings within `Configuration settings` and check the `Enabled?` input. Set the value of `Limit to selector` to `.no-touch`.

### 2.2 Trac configuration 

The Trac instance to be used only requires some small, but important changes. Note that you will require the `TRAC_ADMIN` permission to make theses changes.

#### 2.2.1 Add Element ID field 

With the Trac web interface go to `Admin > Custom Fields` and using the `Add Custom Field` form define a field with the following properties: 
```
Name: element_id
Type: text
Label: Element ID
Format: plain
```

#### 2.2.2 Add Components 

Again, via the Trac web interface go to `Admin > Components` and using the `Add Component` form define components as appropriate for your project. For example, a project with a functional build and two production builds could have three components: `p101`, `m05`, and `m10`. N.B. remove the default components before creating new ones. Typically the default owner should be set to `somebody`. 

Ensure that the `_id` property in `course.json` for each build folder is specified according to the build name and that each corresponds to a component created in the Trac web interface. This allows tickets to be matched to the correct build. 

#### 2.2.3 Create the Adapt/Trac integration report 

This step creates a report that includes all tickets and includes the `keywords` and `element_id` fields. Create a new report called “Adapt/Trac integration” and use the following query: 

```sql
SELECT p.value AS __color__, id AS ticket, summary, component, status, owner, time AS created, changetime AS _changetime, description AS _description, reporter AS _reporter, 

   (CASE WHEN c.value = '0' THEN 'None' ELSE c.value END) AS element_id, 

   status, keywords 

  FROM ticket t 

  LEFT OUTER JOIN ticket_custom c ON (t.id = c.ticket AND c.name = 'element_id') 

  JOIN enum p ON p.name = t.priority AND p.type = 'priority' 

  ORDER BY p.value, milestone, severity, time 
```
N.B. some Trac instances may not have any priorities defined (often the case for client-facing instances). If this is the case then amend the above as follows (otherwise you may find the above report produces nothing): 

```sql
SELECT id AS ticket, summary, component, status, owner, time AS created, changetime AS _changetime, description AS _description, reporter AS _reporter, 

   (CASE WHEN c.value = '0' THEN 'None' ELSE c.value END) AS element_id, 

   status, keywords 

  FROM ticket t 

  LEFT OUTER JOIN ticket_custom c ON (t.id = c.ticket AND c.name = 'element_id') 

  ORDER BY milestone, severity, time
```

#### 2.2.4 Check the report URL

Once the report has been created make a note of the index. In the Trac web interface go to `View Tickets > Available Reports` and look for the index that precedes “Adapt/Trac integration”. In most cases it will be `9`. If it is not, the Trac configuration must be updated.

_instructions for Moodle deployment_
Make a note of the report URL. This URL will be used to configure the `Trac Report URI` field during configuration of the Moodle course (see [link](#23-moodle-configuraton)).

_instructions for legacy deployment_
Within the plugin configuration locate the key `_tracPathReport` and change `report/9` to `report/X`; where `X` is the correct index of the report.

### 2.2.5 Permissions

Via the Trac web interface go to `Admin > Permissions` and using the `Grant Permission` form add the following if it does not yet exist: 
```
Subject: [the username of an account with access to the target Trac instance]
Action: TRAC_ADMIN 
```
### 2.3 Moodle configuration

The Moodle plugin allows each course to be configured to access a single Trac instance. When you create a course keep a note of its unique reference: this is the `idnumber`. You can find this reference by looking at the URL of the course, e.g. `https://clients.kineo.com/chrissteele/course/view.php?id=10`: in this case the `idnumber` is `10`.

Once you have configured the Trac instance that the course will use open the configuration page: `https://clients.kineo.com/[client]/local/trac_api/config.php` and create a new entry by populating the form with the following properties:

- `Unique reference` use the value of `idnumber` as described in [2.3 Moodle configuration](#23-moodle-configuration)
- `Username` the username of an account with `TRAC_ADMIN` privileges on the target Trac instance (see [2.2.5 Permissions](#225-permissions))
- `New Password` the password of the account given in `Username`
- `Trac RPC Endpoint` location of the RPC endpoint (a URL suffixed `/login/rpc` e.g. `https://dev.kineo.com/trac/csdev/login/rpc`)
- `Trac Report URI` the URL described in [2.2.4 Check the report URL](#224-check-the-report-url)
- `Trac Environment Path` the full file system path to the Trac instance (e.g. `/var/www/vhosts/dev.kineo.com/TRAC_INSTANCES/csdev`)

### 3 Misc notes

Early versions of the plugin used a separate file to store configuration data; `trac.json`. This is not suitable for deployment on AAT instances and so the configuration has been moved to `course.json`.

At the time of writing there exists a [Moodle plugin](#23-moodle-configuration) that serves as the backend for client site deployments (`clients.kineo.com`). A legacy backend exists for other deployments which resides on `dev.kineo.com`. This allows the plugin to work from AAT instances and subversion.

The Moodle plugin requires reviewers to possess a valid Moodle login to permit access to Trac data. It further limits access to one Trac instance per course. This allows fine control over which Trac instances can by accessed by clients. The legacy backend is protected by basic auth and requires a valid Kineo login. It is therefore only suitable at present for internal reviews. It is expected that the AAT instances will be modified to allow a similar level of access control in the near future; thus allowing external users to perform reviews.

Trac instances require the [TracXMLRPC](https://trac-hacks.org/wiki/XmlRpcPlugin) and [TracCustomFieldAdmin](https://trac-hacks.org/wiki/CustomFieldAdminPlugin) plugins to be enabled (these should already be enabled via a common configuration).

The plugin has only been tested with Trac 1.0.19.

There are problems with the jQuery.csv library when using Trac 1.4.
