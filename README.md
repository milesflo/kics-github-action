# KICS Github Action ![kics](images/icon-32x32.png) <img src="images/github.png" alt="Github" width="40" height="40">

[![License: GPL-3.0](https://img.shields.io/badge/License-GPL3.0-yellow.svg)](https://www.gnu.org/licenses)
[![Latest Release](https://img.shields.io/github/v/release/checkmarx/kics-github-action)](https://github.com/checkmarx/kics-github-action/releases)
[![Open Issues](https://img.shields.io/github/issues-raw/checkmarx/kics-github-action)](https://github.com/checkmarx/kics-github-action/issues)

## Integrate KICS into your GitHub workflows, using KICS Github Action to make your IaC more secure

**KICS** (pronounced as 'kick-s') or **Kicscan** is an open source solution for static code analysis of Infrastructure as Code.

**K**eeping **I**nfrastructure as **C**ode **S**ecure (in short **KICS**) is a must-have for any cloud native project. With KICS, finding security vulnerabilities, compliance issues, and infrastructure misconfigurations happens early in the development cycle, when fixing these is straightforward and cheap.

It is as simple as running a CLI tool, making it easy to integrate into any project CI.

#### Supported Platforms

<img alt="Terraform" src="images/logo-terraform.png" width="150">&nbsp;&nbsp;&nbsp;
<img alt="Kubernetes" src="images/logo-k8s.png" width="150">&nbsp;&nbsp;&nbsp;
<img alt="Docker" src="images/logo-docker.png" width="150">&nbsp;&nbsp;&nbsp;
<br>
<img alt="CloudFormation" src="images/logo-cf.png" width="150">&nbsp;&nbsp;&nbsp;
<img alt="Ansible" src="images/logo-ansible.png" width="150">&nbsp;&nbsp;&nbsp;
<img alt="Helm" src="images/logo-helm.png" width="61" height="70">&nbsp;&nbsp;&nbsp;


### Please find more info in the official website: <a href="https://kics.io">kics.io</a>

## Inputs

| Variable           | Example Value &nbsp;                    | Description &nbsp;                                               | Type    | Required | Default                                       |
| ------------------ | --------------------------------------- | ---------------------------------------------------------------- | ------- | -------- | --------------------------------------------- |
| path               | terraform                               | path to file or directory to scan                                | String  | Yes      | N/A                                           |
| config_path        | ./kics.config                           | path to configuration file                                       | String  | No       | N/A                                           |
| platform_type      | terraform,ansible                       | case insensitive list of platform types to scan                  | String  | No       | All platforms                                 |
| exclude_paths      | ./shouldNotScan/*,somefile.txt          | exclude paths from scan, supports glob, comma separated list     | String  | No       | N/A                                           |
| exclude_queries    | a227ec01-f97a-4084-91a4-47b350c1db54    | exclude queries by providing the query ID, comma separated list  | String  | No       | N/A                                           |
| exclude_categories | 'Observability,Networking and Firewall' | exclude categories by providing its name, comma separated list   | String  | No       | N/A                                           |
| exclude_results    | 'd4a1fa80-d9d8-450f-87c2-e1f6669c41f8'  | exclude results by providing the similarity ID of a result       | String  | No       | N/A                                           |
| output_formats     | 'json,sarif'                            | formats in which the results report will be exported             | String  | No       | json                                          |
| output_path        | results.json                            | file path to store result in json format                         | String  | No       | N/A                                           |
| payload_path       |                                         | file path to store source internal representation in JSON format | String  | No       | N/A                                           |
| queries            |                                         | path to directory with queries (default "./assets/queries")      | String  | No       | ./assets/queries downloaded with the binaries |
| verbose            | true                                    | verbose scan                                                     | Boolean | No       | false                                         |

## Simple usage example

```yaml
    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
    # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
    - uses: actions/checkout@v2
    # Scan Iac with kics
    - name: run kics Scan
      uses: checkmarx/kics-action@v1.0
      with:
        path: 'terraform'
        output_path: 'results.json'
    # Display the results in json format
    - name: display kics results
      run: |
        cat results.json
```

## Example using docker-runner and SARIF report

checkmarx/kics-action@docker-runner branch runs an alpine based linux container (`checkmarx/kics:nightly-alpine`) that doesn't require downloading kics binaries and queries in the `entrypoint.sh`

```yaml
name: scan with KICS docker-runner

on:
  pull_request:
    branches: [master]

jobs:
  kics-job:
    runs-on: ubuntu-latest
    name: kics-action
    steps:
      - name: Checkout repo
        uses: actions/checkout@v2
      - name: Mkdir results-dir
        # make sure results dir is created
        run: mkdir -p results-dir
      - name: Run KICS Scan with SARIF result
        uses: checkmarx/kics-action@docker-runner
        with:
          path: 'terraform'
          # when provided with a directory on output_path
          # it will generate the specified reports file named 'results.{extension}'
          # in this example it will generate:
          # - results-dir/results.json
          # - results-dir/results.sarif
          output_path: results-dir
          platform_type: terraform
          output_formats: 'json,sarif'
          exclude_paths: "terraform/gcp/big_data.tf,terraform/azure"
          # seek query id in it's metadata.json
          exclude_queries: 0437633b-daa6-4bbc-8526-c0d2443b946e
      - name: Show results
        run: |
          cat results-dir/results.sarif
          cat results-dir/results.json
      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v1
        with:
          sarif_file: results-dir/results.sarif
```
## Example using docker-runner and a config file

Check [configuration file](https://github.com/Checkmarx/kics/blob/master/docs/configuration-file.md) reference for more options.

```yaml
name: scan with KICS using config file

on:
  pull_request:
    branches: [master]

jobs:
  kics-job:
    runs-on: ubuntu-latest
    name: kics-action
    steps:
      - name: Checkout repo
        uses: actions/checkout@v2
      - name: Mkdir results-dir
        # make sure results dir is created
        run: mkdir -p results-dir
      - name: Create config file
        # creating a heredoc config file
        run: |
          cat <<EOF >>kics.config
          {
            "exclude-categories": "Encryption",
            "exclude-paths": "terraform/gcp/big_data.tf,terraform/gcp/gcs.tf",
            "log-file": true,
            "minimal-ui": false,
            "no-color": false,
            "no-progress": true,
            "output-path": "./results-dir",
            "payload-path": "file path to store source internal representation in JSON format",
            "preview-lines": 5,
            "report-formats": "json,sarif",
            "type": "terraform",
            "verbose": true
          }
          EOF
      - name: Run KICS Scan using config
        uses: checkmarx/kics-action@docker-runner
        with:
          path: 'terraform'
          config_path: ./kics.config
      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v1
        with:
          sarif_file: results-dir/results.sarif
```

## How To Contribute

We welcome [issues](https://github.com/checkmarx/kics-github-action/issues) to and [pull requests](https://github.com/checkmarx/kics-github-action/pulls) against this repository!

# License

KICS Github Action

This program is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with this program. If not, see https://www.gnu.org/licenses/.
