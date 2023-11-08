# GTM workspace scheduled deploy

Nel caso tu abbia bisogno di pubblicare un workspace di Google Tag Manager ad un orario preciso, ecco una soluzione per farlo:

## Specs:
- 1 x Service account
- 1 x Cloud Scheduler
- 1 x Pub/Sub 
- 1 x Cloud Functions
- 1 x Google Tag Manager Client-side or Server-side

## Architecture schema

<img width="1274" alt="Screenshot 2023-11-08 alle 13 49 55" src="https://github.com/tommasomoretti/gtm-scheduled-deploy/assets/29273232/b2f5a996-4e5c-4534-a6d2-5228de601d7f">

### Service Account
Crea un service account custom e assegnagli il ruolo di editor del progetto Google Cloud.

### Cloud Pub/Sub
Creare un nuovo argomento Cloud Pub/Sub:
- Nome: deploy-gtm-workspace

### Cloud Scheduler
Crea un nuovo Job di Cloud Scheduler come segue:
- Nome: gtm-scheduled-deploy
- Espressione cron: Data in cui dev'essere eseguito il deploy, per info https://crontab.guru/. Es: 12.30 PM, 25 dec 2023 => 30 12 25 12 *
- Fuso orario: UTC
- Tipo di target: Cloud Pub/Sub
- Argomento Cloud Pub/Sub: gtm-scheduled-deploy
- Attributi del messaggio:
  - account_id: your account id
  - container_id: your container_id
  - workspace_id: your workspace id 

### Cloud Functions
Crea una nuova funzione Gen 1 con trigger Cloud Pub/Sub, selezionando deploy-gtm-workspace come nome argomento.

main.py

```python
from google.oauth2 import service_account
from googleapiclient.discovery import build
import smtplib
from email.mime.text import MIMEText

def deploy_gtm_workspace(event, context):
  print("👉🏻 Let's deploy a GTM workspace")
  
  account_id = event['attributes']['account_id']
  print('  👉🏻 Account: ' + account_id)
  
  container_id = event['attributes']['container_id']
  print('  👉🏻 Container: ' + container_id)
  
  workspace_id = event['attributes']['workspace_id']
  print('  👉🏻 Workspace id: ' + workspace_id)


  # Carica le credenziali dal file JSON scaricato da Google Cloud Console
  service_account_credentials = {YOUR SERVICE ACCOUNT SECRET KEY}

  credentials_get_workspace_info = service_account.Credentials.from_service_account_info(service_account_credentials, scopes=['https://www.googleapis.com/auth/tagmanager.readonly'])
  credentials_create_version = service_account.Credentials.from_service_account_info(service_account_credentials, scopes=['https://www.googleapis.com/auth/tagmanager.edit.containerversions'])
  credentials_publish_version = service_account.Credentials.from_service_account_info(service_account_credentials, scopes=['https://www.googleapis.com/auth/tagmanager.publish'])
  
  # Inizializza il client delle API
  service_get_workspace_info = build('tagmanager', 'v2', credentials=credentials_get_workspace_info)
  service_create_version = build('tagmanager', 'v2', credentials=credentials_create_version)
  service_publish_version = build('tagmanager', 'v2', credentials=credentials_publish_version)

  try:
    # Get workspace name and description
    get_workspace_info = service_get_workspace_info.accounts().containers().workspaces().get(path=f'accounts/{account_id}/containers/{container_id}/workspaces/{workspace_id}').execute()
    workspace_name = get_workspace_info['name']

    if 'description' in get_workspace_info:
      workspace_description = get_workspace_info['description']
    else:
      workspace_description = ''

    workspace_details = {
      "name": workspace_name,
      "notes": workspace_description
    }
    print(f'  👉🏻 Workspace name: {workspace_name}')
    print(f'  👉🏻 Workspace descrpition: {workspace_description}')
    
    # Create a version from workspace
    create_version = service_create_version.accounts().containers().workspaces().create_version(path=f'accounts/{account_id}/containers/{container_id}/workspaces/{workspace_id}', body=workspace_details).execute()
    version_id = create_version['containerVersion']['containerVersionId']
    print(f'👍🏻 Version id {version_id} created from workspace id {workspace_id}.')

    # Publish a version
    publish_version = service_publish_version.accounts().containers().versions().publish(path=f'accounts/{account_id}/containers/{container_id}/versions/{version_id}').execute()
    print(f'👍🏻 Version id {version_id} published.')
    
    # Send email notification
    subject = f"GTM Alert | Workspace {workspace_id} published"
    body = f"All done. New version {version_id} created from workspace {workspace_id}."
    send_email_notification(subject, body)

  except Exception as e:
    print(f'🖕🏻 Errore nella pubblicazione del container: {e}')
    subject = f"GTM Alert | Workspace {workspace_id} not published"
    body = f"Error. Workspace {workspace_id} not exist."
    send_email_notification(subject, body)
    # raise

def send_email_notification(subject, body):
  sender = "worker@gmail.com" # Gmail account that send the email
  recipients = [
    "user1@domain.com", # Email recipient 1
    "user2@domain.com"  # Email recipient 2
  ]
  password = "" # App password

  msg = MIMEText(body)
  msg['Subject'] = subject
  msg['From'] = sender
  msg['To'] = ', '.join(recipients)
  
  with smtplib.SMTP_SSL('smtp.gmail.com', 465) as smtp_server:
    smtp_server.login(sender, password)
    smtp_server.sendmail(sender, recipients, msg.as_string())

  print("👍🏻 Email sent")
```

Requirements.txt

```
requests
google-auth
google-auth-oauthlib
google-api-python-client
```


### Google Tag Manager
Aggiungi il Service Account creato come amministratore del'account.
