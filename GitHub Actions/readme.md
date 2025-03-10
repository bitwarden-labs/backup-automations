# Bitwarden Org Vault Backup GitHub Action

## Overview

This GitHub Action automates the backup of a Bitwarden organization vault and uploads it to an S3-compatible object storage. It runs on every push to the `main` branch and can also be scheduled to run daily.

![Bitwarden Logo](../GitHub%20Actions/img/valut-backup-workflow.svg)

## Prerequisites

Before using this action, ensure you have the following:

- A **Bitwarden account** with API access ([Bitwarden CLI Docs](https://bitwarden.com/help/cli/))
- **Bitwarden Secrets Manager** to securely store and retrieve secrets ([Bitwarden Secrets Manager Docs](https://bitwarden.com/help/secrets-manager/))
- An **S3-compatible object storage** (e.g., AWS S3, [Civo Object Storage](https://www.civo.com/docs/object-stores))
- A **GitHub repository** with Secrets configured for authentication and storage access

### Using Bitwarden Secrets Manager

This workflow retrieves secrets from **Bitwarden Secrets Manager**, ensuring that sensitive data is not stored directly in GitHub Secrets. Instead, GitHub Secrets only store the **Secrets Manager IDs**, which are then used to fetch the actual secrets at runtime.

To use Bitwarden Secrets Manager:
1. **Create an account** and set up Secrets Manager in Bitwarden.
2. **Store the necessary secrets** (e.g., API credentials, encryption password, and S3 storage details).
3. **Retrieve the Secrets Manager IDs** for each secret.
4. **Store these IDs in GitHub Secrets** instead of the actual secret values.
5. **Use the Bitwarden Secrets Manager GitHub Action** to fetch the secret IDs during workflow execution.

For more details, visit the [Bitwarden Secrets Manager documentation](https://bitwarden.com/help/secrets-manager/).

### Using Civo Object Storage

This example workflow uses **Civo Object Storage** as the backup destination. Civo provides an S3-compatible storage solution that is easy to set up and use with `s3cmd`.

#### Setting Up Civo Object Storage

1. **Create a Civo account** at [Civo Cloud](https://www.civo.com/).
2. **Navigate to Object Storage** and create a new bucket.
3. **Generate Access Keys** under the Object Storage settings.
4. **Store the credentials as Bitwarden Secrets**:
   - `BW_SM_BITWARDEN_S3_ACCESS_KEY_ID`
   - `BW_SM_BITWARDEN_S3_SECRET_ACCESS_KEY_ID`
   - `BW_SM_BITWARDEN_S3_ENDPOINT_URL_ID`
   - `BW_SM_BITWARDEN_S3_BUCKET_NAME_ID`


For more details, visit the [Civo Object Storage documentation](https://www.civo.com/docs/object-storage).

### Required GitHub Secrets

Set up the following secrets in your repository, storing only the **Bitwarden Secrets Manager IDs**:

| Secret Name                               | Description                             |
| ----------------------------------------- | --------------------------------------- |
| `BW_SM_TOKEN`                             | Bitwarden Secrets Manager machine API token for authentication with Bitwarden Secrets Manager  |
| `BW_SM_URL`                               | Bitwarden Secrets Manager API base URL                  |
| `BW_SM_BITWARDEN_EMAIL_ID`                | Bitwarden email ID stored in Secrets Manager |
| `BW_SM_BITWARDEN_PASSWORD_ID`             | Bitwarden password ID stored in Secrets Manager |
| `BW_SM_BITWARDEN_CLIENT_ID_ID`            | Bitwarden client ID stored in Secrets Manager |
| `BW_SM_BITWARDEN_CLIENTSECRET_ID`         | Bitwarden client secret ID stored in Secrets Manager |
| `BW_SM_BITWARDEN_S3_ACCESS_KEY_ID`        | S3 Access Key ID stored in Secrets Manager |
| `BW_SM_BITWARDEN_S3_SECRET_ACCESS_KEY_ID` | S3 Secret Access Key ID stored in Secrets Manager |
| `BW_SM_BITWARDEN_S3_ENDPOINT_URL_ID`      | S3 Endpoint URL ID stored in Secrets Manager |
| `BW_SM_BITWARDEN_BW_ORG_ID_ID`            | Bitwarden Organization ID stored in Secrets Manager |
| `BW_SM_BITWARDEN_S3_BUCKET_NAME_ID`       | S3 Bucket Name ID stored in Secrets Manager |
| `BW_SM_BITWARDEN_BW_ENC_PASS_ID`          | Encryption password ID stored in Secrets Manager |

## Usage

Also viewable [here](https://github.com/bitwarden-labs/backup-automations/blob/main/.github/workflows/vault-backup.yml)

### Overview of steps

1. The GitHub Action is triggered, in this example we show both a push to main and also a nightly scheduled task using cron.
2. The Bitwarden Secrets Manager Action is used to connect to Bitwarden Secrets Manager, retrieving the API key to do so from the GitHub Actions secrets.
3. The Bitwarden Secrets Manager Action then retrieves the IDs from GitHub Secrets to retrieve the actual secret values from Bitwarden Secrets Manager.
4. S3CMD is installed.
5. Bitwarden CLI is installed.
6. Log into Bitwarden using the API key method, utilizing the user's client ID and secret (pulled previously from Secrets Manager).
7. Unlock the vault with the user's master password.
8. Export the vault using the Bitwarden CLI, ensuring the export is encrypted using the secret also pulled previously.
9. Upload the exported vault to the configured S3 storage account.
10. Cleanup backups, retaining the last 10 backups in the S3 storage.

### Workflow Configuration

```yaml
name: "Bitwarden Org Vault Backup"
on:
  push:
    branches:
      - main
  # Uncomment this to run daily
  schedule:
    - cron: "0 0 * * *"

jobs:
  backup:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: bitwarden/sm-action@v2
        with:
          access_token: ${{ secrets.BW_SM_TOKEN }}
          base_url: ${{ secrets.BW_SM_URL }}
          secrets: |
            ${{ secrets.BW_SM_BITWARDEN_EMAIL_ID }} > BITWARDEN_EMAIL
            ${{ secrets.BW_SM_BITWARDEN_PASSWORD_ID }} > BITWARDEN_PASSWORD
            ${{ secrets.BW_SM_BITWARDEN_CLIENT_ID_ID }} > BW_CLIENTID
            ${{ secrets.BW_SM_BITWARDEN_CLIENTSECRET_ID }} > BW_CLIENTSECRET
            ${{ secrets.BW_SM_BITWARDEN_S3_ACCESS_KEY_ID }} > S3_ACCESS_KEY_ID
            ${{ secrets.BW_SM_BITWARDEN_S3_SECRET_ACCESS_KEY_ID }} > S3_SECRET_ACCESS_KEY
            ${{ secrets.BW_SM_BITWARDEN_S3_ENDPOINT_URL_ID }} > S3_ENDPOINT_URL
            ${{ secrets.BW_SM_BITWARDEN_BW_ORG_ID_ID }} > BW_ORG_ID
            ${{ secrets.BW_SM_BITWARDEN_S3_BUCKET_NAME_ID }}  > S3_BUCKET_NAME
            ${{ secrets.BW_SM_BITWARDEN_BW_ENC_PASS_ID }}  > ENC_PASS

      - name: "Install S3CMD"
        run: |
          sudo apt-get update
          sudo apt-get install -y s3cmd     

      - name: "💿 Install Bitwarden CLI (Standalone)"
        run: |
          set -e
          curl -L "https://vault.bitwarden.com/download/?app=cli&platform=linux" -o bw.zip
          unzip bw.zip
          sudo mv bw /usr/local/bin/

      - name: "🔐 Log in to Bitwarden CLI with API KEY"
        run: |
          set -e
          bw login --apikey --quiet
        shell: bash

      - name: "🔓 Unlock Vault"
        run: |
          set -e
          SESSION=$(bw unlock --passwordenv BITWARDEN_PASSWORD --raw)
          echo "::add-mask::$SESSION"
          echo "BW_SESSION=$SESSION" >> $GITHUB_ENV
        shell: bash

      - name: "📦 Export Vault"
        run: |
          set -e
          echo "::add-mask::$BW_SESSION"  
          echo "Backing up Bitwarden Vault...."

          # Export the vault using the BW_SESSION from the previous step
          bw export --session "$BW_SESSION" --organizationid $BW_ORG_ID --format encrypted_json --password $ENC_PASS --output vault.json

          # Ensure file exists
          if [ ! -f vault.json ]; then
            echo "❌ Export failed! Vault file not found."
            exit 1
          fi

          echo "✅ Vault exported successfully!"
          echo "Logging out..."
          bw logout
          # Clear BW_SESSION after use
          unset BW_SESSION
        shell: bash

      - name: "☁️ Upload to Civo Object Storage"
        run: |
          set -e  # Stop on first error
          
          # Export S3 credentials as environment variables
          export AWS_ACCESS_KEY_ID="${{ env.S3_ACCESS_KEY_ID }}"
          export AWS_SECRET_ACCESS_KEY="${{ env.S3_SECRET_ACCESS_KEY }}"
          export S3_HOST="${{ env.S3_ENDPOINT_URL }}"
          export S3_BUCKET="${{ env.S3_BUCKET_NAME }}"

          # Generate timestamp
          TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")
          BACKUP_FILE="bitwarden_backup_$TIMESTAMP.json"

          mv vault.json "$BACKUP_FILE"
          
          echo "Uploading vault.json to S3..."
    
          # Use environment variables directly without relying on --no-config
          S3CMD_OPTS="--access_key=$AWS_ACCESS_KEY_ID --secret_key=$AWS_SECRET_ACCESS_KEY --host=$S3_HOST --host-bucket=$S3_HOST"
    
          # Upload the file using s3cmd
          s3cmd $S3CMD_OPTS put "$BACKUP_FILE" s3://$S3_BUCKET/ --acl-private
    
          echo "✅ Upload successful!"
        shell: bash
      
      - name: "🧹 Delete old backups (keep last 10)"
        run: |
          set -e  # Stop on first error
          
          # Export S3 credentials as environment variables
          export AWS_ACCESS_KEY_ID="${{ env.S3_ACCESS_KEY_ID }}"
          export AWS_SECRET_ACCESS_KEY="${{ env.S3_SECRET_ACCESS_KEY }}"
          export S3_HOST="${{ env.S3_ENDPOINT_URL }}"
          export S3_BUCKET="${{ env.S3_BUCKET_NAME }}"
          
          S3CMD_OPTS="--access_key=$AWS_ACCESS_KEY_ID --secret_key=$AWS_SECRET_ACCESS_KEY --host=$S3_HOST --host-bucket=$S3_HOST"
          # List backups and filter filenames matching "bitwarden_backup_"  
          BACKUP_FILES=$(s3cmd $S3CMD_OPTS ls s3://$S3_BUCKET/ | grep 'bitwarden_backup_' | awk '{print $4}' | sort)
          # Count total backups  
          TOTAL_BACKUPS=$(echo "$BACKUP_FILES" | wc -l)
          # Set the number of backups to keep  
          KEEP_COUNT=10
          # Ensure DELE  TE_COUNT is never negative
          DELETE_COUNT=$((TOTAL_BACKUPS - KEEP_COUNT))
          if [ "$DELETE_COUNT" -lt "0" ]; then
              DELETE_COUNT=0
          fi
          echo "📂 Total backups found: $TOTAL_BACKUPS"
          echo "📦 Keeping last $KEEP_COUNT backups"
          # Only print delete message if there are b  ackups to delete
          if [ "$DELETE_COUNT" -gt "0" ]; then
              echo "🗑️  Deleting $DELETE_COUNT old backups..."
              echo "$BACKUP_FILES" | head -n "$DELETE_COUNT" | while read -r line; do
                  echo "❌ Deleting old backup: $line"
                  s3cmd $S3CMD_OPTS del "$line"
              done
              echo "✅ Deleted $DELETE_COUNT old backups!"
          else
              echo "🎉 No old backups to delete!"
          fi
          #   Show how many remain
          echo "📂 Remaining backups: $(s3cmd $S3CMD_OPTS ls s3://$S3_BUCKET/ | grep 'bitwarden_backup_' | wc -l)"
    

```

## Notes

- The workflow exports the Bitwarden vault in **encrypted JSON format**.
- Old backups are deleted, keeping only the last **10 backups**.
- S3CMD is used to upload the vault backup to an **S3-compatible storage**.
- If any step fails, the workflow will **exit immediately**.

## References

- [Bitwarden CLI Documentation](https://bitwarden.com/help/cli/)
- [Bitwarden Secrets Manager Documentation](https://bitwarden.com/help/secrets-manager/)
- [s3cmd Documentation](https://s3tools.org/usage)
- [Civo Object Storage Documentation](https://www.civo.com/docs/object-storage)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)

---

For issues or contributions, open a pull request on your repository. 🚀

