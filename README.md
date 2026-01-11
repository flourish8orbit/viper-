name: WhatsApp Bug & Crash Reporting Bot

on:
  issues:
    types: [opened, edited]
  pull_request:
    types: [opened, synchronize, edited]
  schedule:
    - cron: '0 */6 * * *'  # Run every 6 hours

jobs:
  bug-reporter:
    runs-on: ubuntu-latest
    name: Bug & Crash Report Handler

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install requests pyyaml

      - name: Parse bug/crash reports
        id: parse_issues
        run: |
          python scripts/parse_bug_reports.py
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Send WhatsApp notification
        if: steps.parse_issues.outputs.has_bugs == 'true'
        run: |
          python scripts/whatsapp_notifier.py
        env:
          WHATSAPP_API_URL: ${{ secrets.WHATSAPP_API_URL }}
          WHATSAPP_API_TOKEN: ${{ secrets.WHATSAPP_API_TOKEN }}
          WHATSAPP_PHONE_NUMBER: ${{ secrets.whatsapp-phone-number:$}}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Log report status
        if: always()
        run: |
          echo "Bug reporting workflow completed at $(date -u +'%Y-%m-%d %H:%M:%S UTC')"
          python scripts/log_report_status.py
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Create issue labels
        if: steps.parse_issues.outputs.has_bugs == 'true'
        uses: actions/github-script@v7
        with:
          script: |
            const labels = ['bug-report', 'crash-report', 'critical', 'whatsapp-notified'];
            for (const label of labels) {
              try {
                await github.rest.issues.createLabel({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  name: label,
                  color: label === 'critical' ? 'ff0000' : 'fbca04',
                  description: `Automatically tracked ${label.replace('-', ' ')}`
                });
              } catch (error) {
                if (error.status !== 422) throw error;
              }
            }

      - name: Archive workflow logs
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: bug-bot-logs-${{ github.run_id }}
          path: logs/
          retention-days: 30

  alert-critical:
    runs-on: ubuntu-latest
    name: Critical Alert Handler
    if: contains(github.event.issue.labels.*.name, 'critical') || contains(github.event.pull_request.labels.*.name, 'critical')

    steps:
      - name: Send critical alert
        run: |
          echo "CRITICAL BUG ALERT: Processing critical issue/PR"
        env:
          ISSUE_NUMBER: ${{ github.event.issue.number || github.event.pull_request.number }}
          ISSUE_TITLE: ${{ github.event.issue.title || github.event.pull_request.title }}

      - name: Notify via WhatsApp 814 828 7012
        uses: actions/github-script@v7
        with:
          script: |
            console.log('Critical bug alert would be sent via WhatsApp');

  analytics:
    runs-on: ubuntu-latest
    name: Bug Report Analytics
    if: github.event_name == 'schedule'

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Generate analytics report
        run: |
          python scripts/generate_analytics.py
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Upload analytics
        uses: actions/upload-artifact@v3
        with:
          name: bug-analytics-report-${{ github.run_id }}
          path: reports/analytics.json
          retention-days: 90
