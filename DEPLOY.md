# Staging Deployment Runbook

## Trigger
Every push to `main` runs `test -> build -> deploy`. Pull requests only run `test`.
The `deploy` job starts only if `build` succeeded, and `build` only if `test` succeeded.

## Target
- Platform: Render Web Service, deployed from the GHCR image
  `ghcr.io/hugol4ka/hbtn-devops-pipeline-lab:latest`.
- The `deploy` job calls the Render API to start a new deploy, which pulls the current `latest` image.
- Credentials live in GitHub repository secrets `RENDER_API_KEY` and `RENDER_SERVICE_ID`.
- The public URL lives in the repository variable `STAGING_URL`.

## Database configuration
- Disposable Render PostgreSQL database in the same region as the Web Service.
- The Web Service environment variable `DATABASE_URL` holds the database's internal connection string.
  It is set in the Render dashboard only, never in the repository or the workflow.
- The application runs its migrations on startup.

## Verification
The pipeline retries both checks for a bounded time and fails the job if they do not both return HTTP 200.
To check manually:

    curl -s -o /dev/null -w '%{http_code}\n' "$STAGING_URL/health"   # process liveness
    curl -s -o /dev/null -w '%{http_code}\n' "$STAGING_URL/items"    # API can query PostgreSQL

Both must return `200`.

## Rollback
Every published image also has an immutable tag equal to its commit SHA.
1. Find the last good commit SHA in the Actions history (a green `deploy` run).
2. In Render: Web Service -> Settings -> Image URL, set
   `ghcr.io/hugol4ka/hbtn-devops-pipeline-lab:<full-commit-sha>` and save.
3. Trigger a manual deploy and run the two verification requests above.
4. Once the fix is merged to `main`, set the image URL back to `:latest`.

## Cleanup
When the lab is no longer needed:
1. Delete the Render Web Service and the Render PostgreSQL database.
2. Revoke the Render API key (Account Settings -> API Keys).
3. Delete the GitHub secrets `RENDER_API_KEY`, `RENDER_SERVICE_ID` and the variable `STAGING_URL`.
4. Revoke any registry credential created for Render, if one was used.
5. Decide on the GHCR package visibility: set it back to private, or delete the package.
