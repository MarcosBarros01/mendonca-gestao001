name: CI & Deploy with Vercel + Netlify fallback

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      status: ${{ job.status }}

    steps:
      - uses: actions/checkout@v3
      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      - name: Install
        run: npm ci
      - name: Run tests
        run: npm test -- --coverage --watchAll=false
      - name: Deploy to Vercel (if tests pass)
        if: success()
        uses: amondnet/vercel-action@v20
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          working-directory: ./

  netlify-fallback:
    runs-on: ubuntu-latest
    needs: build
    if: needs.build.result == 'failure'
    steps:
      - uses: actions/checkout@v3
      - name: Install
        run: npm ci
      - name: Build
        run: npm run build
      - name: Deploy to Netlify (fallback)
        uses: nwtgck/actions-netlify@v2.0.0
        with:
          publish-dir: './dist'
          production-deploy: true
          github-token: ${{ secrets.GITHUB_TOKEN }}
          netlify-auth-token: ${{ secrets.NETLIFY_AUTH_TOKEN }}
          site-id: ${{ secrets.NETLIFY_SITE_ID }}

  notify:
    runs-on: ubuntu-latest
    needs: [build, netlify-fallback]
    if: always()
    steps:
      - name: Slack notify (if configured)
        if: ${{ secrets.SLACK_WEBHOOK_URL != '' }}
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            { "text": "*Deploy Mendonça - ${{ github.repository }}*", "attachments": [{"fields":[{"title":"Branch","value":"${{ github.ref_name }}","short":true},{"title":"Build","value":"${{ needs.build.result }}","short":true}]}] }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      - name: Email notify (if configured)
        if: ${{ secrets.MAIL_USERNAME != '' }}
        uses: dawidd6/action-send-mail@v3
        with:
          server_address: smtp.gmail.com
          server_port: 465
          username: ${{ secrets.MAIL_USERNAME }}
          password: ${{ secrets.MAIL_PASSWORD }}
          subject: "CI - Mendonça - Status: ${{ needs.build.result }}"
          to: ${{ secrets.MAIL_TO }}
          from: "CI <${{ secrets.MAIL_USERNAME }}>"
          body: |
            Build: ${{ needs.build.result }}
            Repo: ${{ github.repository }}
            Branch: ${{ github.ref_name }}
