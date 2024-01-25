# Golden VCR

The Golden VCR platform exists for two main purposes:

1. To catalog a collection of VHS tapes and present them on a public-facing website,
   [goldenvcr.com](https://goldenvcr.com), that allows users to browse the tapes and
   _(if logged in with their Twitch account)_ mark certain tapes as favorites, view
   their balance of Golden VCR Fun Points, and use those points to interact with streams

2. To facilitate streams on [twitch.tv/GoldenVCR](https://twitch.tv/GoldenVCR) where we
   choose tapes from the collection and watch them together, using Twitch integration to
   facilitate viewer interactions (and trigger alerts and other effects through the
   GoldenVCR backend) directly via Twitch

The Golden VCR platform consists of several backend services (written in Go) and a
handful of frontend applications (written in TypeScript using Svelte). We run our own
PostgreSQL and RabbitMQ servers as part of the backend, and our backend applications
interact with several external APIs, including Google Sheets, Twitch, Discord, and
OpenAI.

**NOTICE:** All code in this GitHub organization is provided without a license. These
repositories are public for educational purposes only.

## Repositories

| Core Platform                                                    | Backend Services                                       | Frontend Apps                                              |
| ---------------------------------------------------------------- | ------------------------------------------------------ | ---------------------------------------------------------- |
| [**.github**](https://github.com/golden-vcr/.github)             | [**auth**](https://github.com/golden-vcr/auth)         | [**frontend**](https://github.com/golden-vcr/frontend)     |
| [**terraform**](https://github.com/golden-vcr/terraform)         | [**ledger**](https://github.com/golden-vcr/ledger)     | [**graphics**](https://github.com/golden-vcr/graphics)     |
| [**image-tools**](https://github.com/golden-vcr/image-tools)     | [**tapes**](https://github.com/golden-vcr/tapes)       | [**extensions**](https://github.com/golden-vcr/extensions) |
| [**video-tools**](https://github.com/golden-vcr/video-tools)     | [**showtime**](https://github.com/golden-vcr/showtime) |                                                            |
| [**server-common**](https://github.com/golden-vcr/server-common) | [**hooks**](https://github.com/golden-vcr/hooks)       |                                                            |
| [**schemas**](https://github.com/golden-vcr/schemas)             | [**chatbot**](https://github.com/golden-vcr/chatbot)   |                                                            |
|                                                                  | [**dispatch**](https://github.com/golden-vcr/dispatch) |                                                            |
|                                                                  | [**dynamo**](https://github.com/golden-vcr/dynamo)     |                                                            |
|                                                                  | [alerts](https://github.com/golden-vcr/alerts)         |                                                            |

## Core Platform

The backend runs on DigitalOcean and is manually deployed to a series of droplets. This
deployment process involves connecting to the droplet via SSH, pulling the latest code
from GitHub, building the requisite binaries, and running them. Structured log output
for all applications is piped to `/var/log/gvcr/*.log`, and debugging/monitoring are
simply achieved via SSH. We rely on this basic, manual process largely because managed
database servers and Kubernetes clusters cost too much money to be a wise investment at
our current scale.

The deployment process is automated via Bash scripts and is designed to be largely
idempotent: after spinning up all required infrastructure with `terraform apply`, you
can bootstrap the entire backend by simply running a deployment script. Running the same
script thereafter will ensure that the backend is running the latest builds of all
applications. You can return to a clean slate at any point by rebuilding all
DigitalOcean droplets (reverting them to a clean Ubuntu image) and re-running the
deployment script.

The frontend applications, once bundled, are deployed to an S3-compatible bucket in
DigitalOcean Spaces: e.g. the main frontend application is served from
`https://golden-vcr-frontend.nyc3.digitaloceanspaces.com/`.

All requests to `goldenvcr.com` go through Cloudflare, which sends them on to our main
HTTP-server droplet, where they're handled by an NGINX server:

- Requests to `goldenvcr.com/api/<service>/...` are reverse-proxied to the appropriate
  HTTP server application (e.g. `GET /api/tapes/catalog` hits
  `http://localhost:5000/catalog` on the droplet)

- Other requests are directed to the appropriate frontend app, served from a Spaces
  bucket

The platform as a whole is supported via a number of different repos, including:

- [**terraform**](https://github.com/golden-vcr/terraform), which includes declarative
  configuration of all backend resources, documentation of all manual setup steps not
  managed via terraform, NGINX configuration, and scripts for deploying and managing the
  backend

- [**image-tools**](https://github.com/golden-vcr/image-tools), a collection of scripts
  for offline processing of images scanned from VHS tapes, which also handles syncing
  tape images to an S3-compatible bucket for use in the platform

- [**video-tools**](https://github.com/golden-vcr/image-tools), a collection of scripts
  to aid in capturing video from a VCR during streams using OBS, then later processing
  and editing those captures with DaVinci Resolve

- [**server-common**](https://github.com/golden-vcr/server-common), a Go library
  defining shared functionality used across backend services

- [**schemas**](https://github.com/golden-vcr/schemas), data type definitions for the
  events that propagate through the backend in message queues, allowing services to
  coordinate complex interactions without being directly coupled

## Backend Services

The backend codebase is written entirely in Go, and it is compiled to (and deployed as)
a handful of HTTP server applications, event consumers, and command-line utilities.

The backend can be roughly split into two categories: essential services used outside of
streams, and services that facilitate interactions during streams. The former category
of core services includes:

- [**auth**](https://github.com/golden-vcr/auth), which permits users to log in to the
  platform via Twitch, allows API requests to be authorized with the resulting Twitch
  User Access Tokens, and issues JWTs allowing internal services to act with authority
  on a user's behalf

- [**ledger**](https://github.com/golden-vcr/ledger), which tracks each user's balance
  of Golden VCR Fun Points and allows transactions to be initiated using those points

- [**tapes**](https://github.com/golden-vcr/tapes), which ingests data from a Google
  spreadsheet and an S3 bucket containing scanned images, in order to populate a
  database of tapes; and which serves information about those tapes to client
  applications

The remaining services deal with making live streams happen and allowing interesting
things to happen during those streams. Currenly, most of that functionality is
concentrated in a single codebase:

- [**showtime**](https://github.com/golden-vcr/showtime), which encapsulates everything
  that happens during streams, including tracking broadcast history, responding to
  Twitch events, serving chat and alert data to frontend apps, and legacy image
  generation functionality for alerts

Much of that functionality is currently being factored out and reworked into a
collection of smaller services, each with a more limited scope. Those new services
include:

- [**hooks**](https://github.com/golden-vcr/hooks), which manages Twitch EventSub
  subscriptions and handles webhook requests initiated by Twitch in response to relevant
  events occurring on the channel

- [**chatbot**](https://github.com/golden-vcr/chatbot), which remains resident in Twitch
  chat in order to respond to IRC events and handle commands encoded in viewer messages

- [**dispatch**](https://github.com/golden-vcr/dispatch), which responds to events
  instigated by Twitch in order to kick off whatever backend events need to follow, such
  as crediting fun points, updating viewer state, producing alerts, or initiating image
  generation for alerts

- [**dynamo**](https://github.com/golden-vcr/dynamo), which facilitates procedural
  generation and processing of images and other assets in response to viewer prompts

- [**alerts**](https://github.com/golden-vcr/alerts), which ultimately serves the final
  alerts that should be displayed onscreen during streams (via the
  [**graphics**](https://github.com/golden-vcr/graphics) app rendered in OBS)

For more information on how these newer services fit together, see the overview of event
types and message queues described in the README for the
[**schemas**](https://github.com/golden-vcr/schemas) repository.

All services are deisgned to be able to run locally with minimal effort: each repo's
README contains instructions for populating an `.env` file with the necessary
configuration details (from terraform state etc.); and then each process can simply be
run with `go run cmd/<entrypoint>/main.go`, with server processes accessible via
`http://localhost:<service-specific-port>`.

## Frontend Apps

The frontend codebase is written in TypeScript, using the Svelte compiler and the Vite
build system. Once each app is bundled to a JavaScript redistributable, it's uploaded to
an S3-compatible bucket in DigitalOcean spaces.

Frontend repos include:

- [**frontend**](https://github.com/golden-vcr/frontend), the main frontend app served
  to users at https://goldenvcr.com, which allows any user to browse the collection of
  tapes, and which allows logged-in users to mark favorite tapes, submit alerts, etc.

- [**graphics**](https://github.com/golden-vcr/graphics), a separate frontend app that
  implements the onscreen graphics displayed during streams, rendered in an OBS Browser
  Source

- [**extensions**](https://github.com/golden-vcr/extensions), frontend apps loaded by
  Twitch and displayed to users within the stream viewing UX (currently unused / on
  hold)

Frontend apps that are deployed to `https://goldenvcr.com` make requests against the
backend using `/api/...` URLs: in production, these naturally hit the service APIs
deployed to the production backend, on the same origin. For local development, the dev
server is configured to proxy `/api/...` requests to `https://goldenvcr.com` for the
same effect, although the proxy can be pointed to locally-running APIs if desired.

## Platform Improvements

This is a passion project, with a single developer, that receives relatively little
traffic and generates relatively little revenue. There are many things that could be
done to improve Golden VCR's production suitability and/or security posture, including
_(roughly ordered from cheapest to most expensive)_:

- Refrain from running applications as root
- Configure a VPC and run a separate internet gateway
- Restrict SSH access and run a separate bastion host
- Run HyperDX to aggregate logs and faciliate easier monitoring/tracing
- Build backend applications to container images with Ko and push them to a registry,
  rather than cloning from GitHub, building from source, and running compiled binaries
  in situ
- Use docker compose to configure and run the backend stack, rather than relying on
  custom Bash scripts to handle deployment and updates
- Use Kubernetes resources to describe deployments, services, ingresses, etc. for
  backend applications, and run k0s to manage deployments with proper health probes,
  management of secrets, etc.
- Provision managed servers for PostgreSQL, RabbitMQ, etc.; rather than running them in
  droplets and handling configuration and backups via custom scripts
- Run the backend in a managed, full-scale Kubernetes cluster

## Further Reading

If you're interested in learning more about how the Golden VCR platform fits together,
here are a few good places to start reading next:

- The README for the [**terraform**](https://github.com/golden-vcr/terraform) repo
  describes the project's infrastructure and setup requirements in more detail, and
  there are additional documents describing each of the external, third-party
  dependencies.

- The README for the [**schemas**](https://github.com/golden-vcr/schemas) repo provides
  an overview of the message queues and event types used to propagate events through the
  backend: it's a particularly good starting point for understanding how viewer
  interactions on Twitch end up producing onscreen effects in OBS.

- The [**image-tools**](https://github.com/golden-vcr/image-tools) and
  [**video-tools**](https://github.com/golden-vcr/video-tools) READMEs explain the
  offline processes used to ingest new tapes into the collection and deal with video
  that's captured from those tapes.

Beyond that, feel free to crack into the README and source code of a repository that
piques your interest: most of the code is fairly well-commented.
