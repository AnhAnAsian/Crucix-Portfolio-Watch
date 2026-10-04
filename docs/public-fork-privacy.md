# Privacy in this public fork

Personal `watchlist.json`, `.env`, generated `runs/` and `output/` files, and
logs are excluded from Git. Keep real credentials in local environment files
or GitHub Actions Secrets; use sample data in committed documentation.

The Daily Analysis workflow runs on its existing schedule but publishes only
success or failure. Both standard output and standard error from the analysis
are saved to a temporary file on the runner. Analysis files and logs are not
uploaded as artifacts and disappear when the hosted runner is discarded.
To diagnose a failure, run `npm run brief:save` locally with your private
configuration.

Git exclusions do not remove previously committed data, and these workflow
changes do not remove historical Actions logs or files that were downloaded.
Do not enable debug tracing or add analysis output to public workflow logs.
