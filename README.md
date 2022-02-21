# NHLBI BioData Catalyst Tracker

> A web-based data tracker for submission process

## Getting started

### Prerequisites

- **[Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)**
- **[Docker](https://www.docker.com/get-started)**
- **[Cloud SDK CLI](https://cloud.google.com/sdk/gcloud)**
- **[SendGrid](https://docs.sendgrid.com/for-developers/sending-email/api-getting-started)**

### Environment variables

For local development, the `api` directory should have an `.env` file with the following:

| name                    | value      | description                                                   |
| ----------------------- | ---------- | ------------------------------------------------------------- |
| DJANGO_LOG_LEVEL        |            | Set to `DEBUG` for local dev and `INFO` for prod              |
| GOOGLE_CLOUD_PROJECT    |            | The Project ID for GCP                                        |
| SECRET_KEY              |            | The `SECRET_KEY` generated by Django                          |
| POSTGRES_DB             | `tickets`  | The Postgres database name                                    |
| POSTGRES_USER           | `postgres` | The username of the Postgres User                             |
| POSTGRES_PASSWORD       |            | A (secure) password for the Postgres User                     |
| POSTGRES_HOST           |            | The external IP for the Compute Engine instance with Postgres |
| POSTGRES_PORT           | `5432`     | The port for the Postgres Database                            |
| SENDGRID_API_KEY        |            | The API key for SendGrid                                      |
| SENDGRID_ADMIN_EMAIL    |            | The admin group email                                         |
| SENDGRID_NO_REPLY_EMAIL |            | The email address to use as the sender                        |
| GOOGLE_CLIENT_ID        |            | The client ID for Google OAuth2                               |
| GOOGLE_CLIENT_SECRET    |            | The client secret for Google OAuth2                           |

> NOTE: [Unless a static IP is assigned](/ansible/README.md#Reserving-a-Static-IP), you must update the `POSTGRES_HOST` variable each time the VM is restarted

### The Postgres Database

Due to the complexity of the setup, we have included some Ansible Playbooks to assist you.
You can find a detailed writeup in the [`ansible`](/ansible) directory

> NOTE: This is required even for local development

### The Django Server

Docker Compose should start a Django server on [`http://0.0.0.0:8000/`](http://0.0.0.0:8000/).
The server uses the `env` file for configuration

> NOTE: Due to Google's OAuth2 requirements, you must use [`http://localhost:8000/`](http://localhost:8000/) to login

### Django Social Auth

We currently use [Google OAuth2 for the authentication](https://django-allauth.readthedocs.io/en/latest/providers.html#google).
You will need to [set up a OAuth consent screen](https://developers.google.com/workspace/guides/configure-oauth-consent).
You must also [set up a OAuth client ID credential](https://developers.google.com/workspace/guides/create-credentials#oauth-client-id)

> NOTE: The `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` variables should be set in the `.env` file

### Admin Panel

NHLBI Admins are able to view all tickets, but Data Custodians are only able to view their own tickets.
In the code, Admins are `staff`.
To access the Admin panel visit [`http://localhost:8000/admin`](http://localhost:8000/admin) as a `superuser` member

Go to the "Tracker/User" panel and select the user you want to grant admin permissions.
You should only need to check the `Is staff` permission for HNLBI Admins.
If you would like to allow that user to access the Admin panel, check the `Is superuser` permission as well

> NOTE: Make sure you save your changes at the bottom

### Docker Compose

Docker is used to standardize the development environment

```
docker-compose up --build
```

> NOTE: This may take a several minutes

You should only need to build this once (or when you make changes to the `Dockerfile` or `docker-compose.yml`).
Any subsequent runs do not require the `--build` flag:

```
docker-compose up
```

## Deployment

We will be using [this guide to deploy to App Engine](https://cloud.google.com/python/django/appengine#macos-64-bit)

### [Permissions](https://cloud.google.com/iam/docs/understanding-roles#predefined)

- [`roles/appengine.appAdmin`](https://cloud.google.com/iam/docs/understanding-roles#app-engine-roles)
- [`roles/cloudbuild.integrationsOwner`](https://cloud.google.com/iam/docs/understanding-roles#cloud-build-roles)
- [`roles/cloudbuild.builds.editor`](https://cloud.google.com/build/docs/iam-roles-permissions#predefined_roles)
- [`roles/secretmanager.admin`](https://cloud.google.com/iam/docs/understanding-roles#secret-manager-roles)
- [`roles/iam.serviceAccountAdmin`](https://cloud.google.com/iam/docs/understanding-roles#service-accounts-roles)
- [`roles/serviceusage.serviceUsageAdmin`](https://cloud.google.com/iam/docs/understanding-roles#service-usage-roles)
- [`roles/storage.admin`](https://cloud.google.com/iam/docs/understanding-roles#cloud-storage-roles)

> NOTE: You will also need to grant yourself the [`roles/iam.serviceAccountUser`](https://cloud.google.com/iam/docs/understanding-roles#service-accounts-roles) on the `App Engine default service account`

#### GitHub Secrets

For your convenience, this repo contains automatic deployment scripts.
These scripts will run depending on different triggers as detailed in the [`.github/workflows/`](.github/workflows/) directory.
To utilize these scripts, you will need to create a GitHub Secret with all the required environment variables listed above

> NOTE: The rest of this README will assume that you have already created the GitHub Secrets above

### Instructions

**Ensure all prior setup is complete before continuing**

To prepare the project for deployment, run the following commands in the `api` directory:

```
python manage.py collectstatic --noinput
python manage.py makemigrations
python manage.py migrate
```

> NOTE: The `collectstatic` command is not required if you are using the GitHub Actions workflow

#### Quay

This repo has been configured to use GitHub Actions to build and push images to Quay on pushes to the `main` branch.
Specifically, the `Dockerfile` in the `api` directory will be used to build the image.
The image will be pushed to the [`nimbusinformatics/bdcat-data-tracker`](https://quay.io/repository/nimbusinformatics/bdcat-data-tracker) repository on Quay.
The image will be named `quay.io/nimbusinformatics/bdcat-data-tracker` and two tags: `latest` and a shortened commit hash.
Include the following in your GitHub Secrets:

| name                 | description                             |
| -------------------- | --------------------------------------- |
| QUAY_NIMBUS_USERNAME | A username with access to the Quay repo |
| QUAY_NIMBUS_PASSWORD | The password of the Quay user           |

Information on how to pull and run the image can be found on the [repo itself](https://quay.io/repository/nimbusinformatics/bdcat-data-tracker?tab=info)

> NOTE: Robot accounts are the preferred method of pushing images to Quay.
> These accounts are usually in the format: `<repo-name+<robot-name>`

#### Google Cloud

##### [Secrets Manager](https://cloud.google.com/python/django/appengine#create-django-environment-file-as-a-secret)

In GCP Secret Manager, you must create a secret called `django_settings`.
You can either upload the `.env` file or paste the secrets in there

> NOTE: If the Compute Engine instance restarts and the IP changes, you must update the `POSTGRES_HOST` variable

##### App Engine

This repo has been configured to use GitHub Actions to deploy to App Engine on pushes to the `main` branch.
For this to work, you must have GitHub Secrets with the following:

| name       | description                                         |
| ---------- | --------------------------------------------------- |
| PROJECT_ID | The Project ID for GCP                              |
| GCP_SA_KEY | The full service account key json exported from GCP |

Alternatively, can navigate to the `api` directory and run:

```
gcloud app deploy
```

#### SendGrid

You will also need to create your own dynamic templates and copy the `TEMPLATE_ID` and assign them into the correct variables in the [`mail.py`](/api/tracker/templates) file

The dynamic templates use handlebars syntax.
These must be included in the dynamic template you make on SendGrid.
These variables are listed under the `self.dynamic_template_data` dictionary of the [`mail.py`](/api/tracker/mail.py) file.
Some templates have been provided for you, but you must create your own if you want to make edits

> NOTE: You will need to [verify the sender identity in SendGrid]("https://docs.sendgrid.com/for-developers/sending-email/sender-identity") for the `SENDGRID_ADMIN_EMAIL`
