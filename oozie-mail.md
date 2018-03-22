# Oozie - Sample Mail Workflow
Send a mail with Oozie!

## General description
  The main idea of this document is to provide the administrator a simple guide to run a simple Oozie Workflow. The example workflow will send a mail to the recievers placed in to and cc fields. Subject and body may also be specified. Oozie mail configuration steps will also be described.

## Configure SMTP in Oozie
  The `oozie-site.xml` file must contain following entries to be able to send mails:
  ```
  oozie.email.from.address = sender.address@domain.you.use
  oozie.email.smtp.auth = false
  oozie.email.smtp.host = smtp.you.use.fqdn
  oozie.email.smtp.port = 25
  ```
  Remember to restart the Oozie server after performing the change.

## Access Oozie WebUI
  The Oozie WebUI rqeuires Kerberos ticket to access. Web browsers are able to forward the ticket, but must be configured so. An example Firefox configuration will be presented.

  1. Open Firefox browser.
  2. Open new tab, type `about:config` in the address bar and open it.
  3. Set following values to listed properties:
    ```
    network.negotiate-auth.trusted-uris = .OOZIE.DOMAIN.YOU.USE
    network.negotiate-auth.delegation-uris = .OOZIE.DOMAIN.YOU.USE
    network.auth.use-sspi = false
    ```
  4. Restart Firefox browser.
  5. Login to Ambari with your LDAP accout and open Oozie service.
  6. Go to Oozie WebUI with Quick Links.
  7. You should be able to see all running Oozie workflows.

## Workflow itself
  The Oozie Workflow job must be composed of two files:
  - job.properties

    ```
    nameNode=hdfs://active.namenode.you.use:8020
    jobTracker=active.resource.manager:8050
    queueName=default
    examplesRoot=examples
    oozie.wf.application.path=${nameNode}/user/${user.name}/${examplesRoot}/apps/mail
    ```

    - nameNode - Active NameNode hostname and port, hdfs protocol must be specified
    - jobTracker - ResourceManager hosntame and port
    - queueName - default will work always
    - examplesRoot - root directory, just a variable
    - oozie.wf.application.path - working directory of the Oozie job

      Job properties must be stored locally, not on HDFS.


  - workflow.xml

    ```
    <workflow-app name="mail-wf" xmlns="uri:oozie:workflow:0.1">
    <start to="mail"/>
      <action name="mail">
        <email xmlns="uri:oozie:email-action:0.1">
          <to>somebody@hpe.com</to>
          <cc>other.somebody@hpe.com</cc>
          <subject>Oozie test mail</subject>
          <body>This is a test mail from Oozie DEV.</body>
        </email>
        <ok to="end"/>
        <error to="fail"/>
      </action>

      <kill name="fail">
          <message>MAIL action failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
      </kill>

      <end name="end"/>
    </workflow-app>
    ```

    The path from `oozie.wf.application.path` must exist on HDFS and the workflow.xml file must be placed on HDFS this directory.

## Running the job
  Connect to the server with Oozie client. Ensure you have Kerberos ticket with `klist`.
  Run the Oozie job with the following command:
  ```
  oozie job -oozie http://oozie.server.you.use:11000/oozie -config /path/to/job.properties -run
  ```
