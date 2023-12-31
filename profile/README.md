# Golden VCR

Source code for [goldenvcr.com](http://goldenvcr.com/).

All code in this GitHub organization is provided without a license. These repositories
are public for educational purposes only.

## [**terraform**](https://github.com/golden-vcr/terraform)

Declarative configuration for backend infrastructure.
  
- README explains the initial account setup for various providers (DigitalOcean,
  Cloudflare, Google) and provides instructions for configuring a workspace from which
  `terraform apply` can be run to provision the resources required to run the
  Golden VCR webapp.

- Currently, the backend is run on a single DigitalOcean droplet, and deployment is a
  manual process. The terraform repo also contains an
  [`init-server.sh`](https://github.com/golden-vcr/terraform/blob/main/init-server.sh)
  script which is used to deploy the latest builds to the server.

## [**image-tools**](https://github.com/golden-vcr/image-tools)

A collection of scripts used locally to facilitate processing of new VHS tapes as
they're added to the Golden VCR library.

- README contains step-by-step instructions for cataloguing new tapes, scanning images,
  and processing and uploading those images to an S3-compatible bucket in DigitalOcean
  Spaces.

- Tapes are assigned a numeric ID and logged in
  [a Google Spreadsheet](https://docs.google.com/spreadsheets/d/1cR9Lbw9_VGQcEn8eGD2b5MwGRGzKugKZ9PVFkrqmA7k/edit).

- [`renumber.sh`](https://github.com/golden-vcr/image-tools/blob/main/renumber.sh)
  quickly renames newly-scanned images to adhere to the desired naming convention.

- [`crop.py`](https://github.com/golden-vcr/image-tools/blob/main/crop.py) allows
  the user to quickly crop each scanned image.

- [`upload.py`](https://github.com/golden-vcr/image-tools/blob/main/upload.py) syncs
  scanned and cropped images to the remote bucket.

## [**video-tools**](https://github.com/golden-vcr/video-tools)

Scripts and tools used to capture and record video footage in a way that's suitable for
simultaneous streaming to Twitch.

- We use a self-contained, portable installation of OBS that's dedicated solely to
  capturing and recording VHS footage, with upscaling and deinterlacing.
  [`install-obs.sh`](https://github.com/golden-vcr/video-tools/blob/main/install-obs.sh)
  installs OBS.

- Machine-specific OBS configurations are versioned in this repo as well: the install
  script will activate the appropriate config by symlinking it as the main obs config
  directory. See [`obs/.gitignore`](https://github.com/golden-vcr/video-tools/blob/main/obs/.gitignore)
  for more details on how these files are versioned.

- README explains how to configure a fresh installation of OBS for capturing VHS
  footage. Configuration files can be added to source control via
  [`bootstrap-obs-config.sh`](https://github.com/golden-vcr/video-tools/blob/main/bootstrap-obs-config.sh).

## [**server-common**](https://github.com/golden-vcr/server-common)

Library containing common code that's shared across multiple backend server applications
(e.g. **auth**, **tapes**, **showtime**).

## [**auth**](https://github.com/golden-vcr/auth)

Backend API that allows users to authenticate with the Golden VCR app by logging in with
Twitch.

- API spec: https://golden-vcr.github.io/auth/

## [**ledger**](https://github.com/golden-vcr/ledger)

Backend API that keeps track of each user's balance of **Golden VCR Fun Points** and
allows transactions to be initiated using those points.

- API spec: https://golden-vcr.github.io/ledger/

## [**tapes**](https://github.com/golden-vcr/tapes)

Backend API that provides the webapp with information about the tapes available in the
Golden VCR library.

- API spec: https://golden-vcr.github.io/tapes/

- The application pulls tape metadata from the Google Sheets API, and it uses the S3
  API to check which images are available in the DigitalOcean Spaces bucket for each
  tape. These results are cached in-memory.

- The application exposes an API, intended for the frontend app to consume, that
  collects summarized information about each tape.

- Usage: `curl https://goldenvcr.com/api/tapes`

## [**showtime**](https://github.com/golden-vcr/showtime)

Backend API that supports Twitch EventSub notifications to facilitate interop between
the Golden VCR app and live streams occurring on Twitch.

- API spec: https://golden-vcr.github.io/showtime/

- Usage: `curl https://goldenvcr.com/api/showtime/status`

## [**frontend**](https://github.com/golden-vcr/frontend)

Frontend webapp, built using Vite and Svelte.

- `npm run dev` will proxy api requests to `goldenvcr.com` by default, allowing you
  to run development builds against live data.

- Deployment is currently a manual process: `rm -rf dist && npm build`, then upload
  to DigitalOcean Spaces bucket. Automated deployment and better support for
  CDN/caching/etc. TBD.

## [**graphics**](https://github.com/golden-vcr/graphics)

Separate frontend codebase for non-interactive overlays that render in OBS while
streaming, in order to embed graphics for alerts etc. into the stream.

## [**extensions**](https://github.com/golden-vcr/extensions)

Frontend apps, also built using Svelte, that render within Twitch to provide extra
information and interactivity to viewers during streams.
