## Usage
Title has to be changed
Everything else should work
```yaml
- name: Send MS Teams Notification
  if: ${{ failure() }}
  uses: EODC-GmbH/github-actions/.github/actions/teams-notification@teams-notifications
  with:
    webhook_url: ${{ secrets.TEAMS_WEBHOOK_URL }}
    title: 'Data moving failed'
    message: 'Workflow run has failed:'
    url: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
    repo_name: ${{ github.repository }}
```


## License
This action is distributed under the terms of the repository’s LICENSE.
This action uses a modified version of haroldelopez's `ms-teams-notification-action` which is distributed under the MIT license. See the LICENSE file in this directory.
