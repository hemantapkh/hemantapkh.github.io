---
title: How We Migrated Our Internal Python Packages From GitLab to Gcloud Artifacts
description: This blog shares the journey of our migration from GitLab registry to Google Cloud Artifacts
author: hemantapkh
date: 2024-08-25 15:48:00 +0000
categories: [DevOps]
tags: [docsumo, gcloud, gitlab, ci/cd]
pin: false
math: false
mermaid: false
image:
  path: /assets/img/posts/migrating-python-packages-from-gitlab-to-gcloud/thumbnail.png
---


At [Docsumo](https://docsumo.com), we recently transitioned our version control system from GitLab to GitHub. One challenge that emerged during this migration was the lack of native support for [Python packages in GitHub](https://github.com/orgs/community/discussions/8542). To address this, we opted to move our internal packages to Google Cloud Artifacts.

Here are the detailed steps I followed to successfully migrate around 250 versions of internal packages from GitLab python registry to Google Cloud Artifacts.

## Create a python registry

First, create a Python registry on Google Cloud Artifacts. You can do this using the [Google Cloud Console interface](https://console.cloud.google.com/artifacts) by selecting `python` as the format and adjusting other options as required.

Alternatively, you can create the registry using the following command:

```bash
gcloud artifacts repositories create python-packages \
    --repository-format=python \
    --location=us \
    --description="Python package repository"
```

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> You can change the repository name from `python-packages` to anything you prefer and adjust other settings as needed.
{: .prompt-tip }

After creating the repository, your repository URL will look like this:

```text
https://<PROJECT_LOCATION>-python.pkg.dev/<PROJECT_ID>/<REGISTRY_NAME>/
```

You can also copy the repository URL from the Google Cloud Console interface.

![IMAGE](/assets/img/posts/migrating-python-packages-from-gitlab-to-gcloud/copy-repo-url.png)

Next, set the repository URL in an environment variable, as it will be required later to upload the packages.

```bash
export REPO_URL=<REPOSITORY_URL>
```

## Download the Packages

The next step is to download the packages from GitLab. Below is the python script that I used. 

Set the `GITLAB_USERNAME`, `GITLAB_TOKEN` and `GITLAB_ORG_NAME` environment variables.


After executing the script, all of your packages will be downloaded to the specified directory as set in the script.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> You can generate a GitLab token [here](https://gitlab.com/-/user_settings/personal_access_tokens); make sure to give the token `read_registry` permission.
{: .prompt-tip }

```python
import requests
import os

GITLAB_USERNAME = os.environ.get("GITLAB_USER_NAME")
GITLAB_TOKEN = os.environ.get("GITLAB_TOKEN")
GITLAB_ORG_NAME = os.environ.get("GITLAB_ORG_NAME")
DOWNLOAD_DIR = "packages"

def get_all_group_packages(group: str, token: str):
    """Use Gitlab api to get all pakages and their metadata"""
    url =f"https://gitlab.com/api/v4/groups/{group}/packages"
    params = {
        "per_page": 400,
        "page": 0,
        "exclude_subgroups": False,
        "order_by": "project_path"
    }

    headers = {
        "PRIVATE-TOKEN": token
    }

    while True:
        response = requests.get(url, headers=headers, params=params).json()

        if not response:
            break

        for package in response:
            yield package

        params["page"] += 1

os.makedirs(DOWNLOAD_DIR, exist_ok=True)

for package in get_all_group_packages(GITLAB_ORG_NAME, GITLAB_TOKEN):
    package_name = package.get("name")+'=='+package.get("version")
    print(f"Downloading {package_name}")
    package_path = package.get("_links",{}).get("delete_api_path")
    package_path = package_path.replace('https://','').rsplit('/', 1)[0]

    index_url = f"https://{GITLAB_USERNAME}:{GITLAB_TOKEN}@{package_path}/pypi/simple"

    command = f"pip download {package_name} --no-deps --no-build-isolation --dest {DOWNLOAD_DIR} --index-url {index_url}"
    os.system(command)
```
{: file='gitlab-packages-downloader.py'}

## Upload the packages

With all package wheels downloaded, the next step is to upload them to the newly created Google Cloud Artifact Registry.

Start by installing the required tools:

```bash
pip install twine keyrings.google-artifactregistry-auth
```

Then, navigate to the directory where the packages are downloaded and use twine to upload them to Google Cloud.

```bash
twine upload --repository-url $REPO_URL *.whl --verbose
```

After running the above command, the packages should start uploading to Google Cloud Artifact Registry.

I hope this guide helps you in making a similar transition smoothly. Feel free to reach out if you have any questions or need any assistance!
