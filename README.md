# n8n-cookbook

A cookbook of reusable [n8n](https://n8n.io) automation templates.

## Layout

```
templates/
  <template-name>/
    workflow.json   # exported n8n workflow (credentials stripped)
    README.md       # what it does, required credentials, setup steps
```

- One folder per template, named in `kebab-case`.
- `templates/_example-template/` shows the expected shape. Copy it to start a new one.

## Templates

### Documents and data

- [Invoice and receipt PDFs to Google Sheets](templates/invoice-receipt-pdf-to-sheets/): logs invoice and receipt data from Gmail PDF attachments.
- [Recurring report from Google Sheets](templates/recurring-sheets-report/): weekly or monthly report emailed from a sheet.
- [Form submissions to documents](templates/form-to-document/): turns a form into a quote, contract or certificate PDF.

## Ideas

See [docs/automation-ideas.md](docs/automation-ideas.md) for a backlog of office automations that would make good templates, with the ones already built marked.

## Adding a template

1. In n8n, open the workflow and choose **Download** (or select all nodes and copy).
2. Save it as `templates/<template-name>/workflow.json`.
3. Check the JSON contains no secrets: no API keys, tokens, passwords, or webhook URLs with secrets in them. n8n exports only credential *references* (id + name), which is fine, but values typed directly into node parameters are not.
4. Fill in the template's `README.md`.

## Importing a template

In n8n: **Workflows → Import from File**, pick the `workflow.json`, then connect your own credentials on each node that needs them.

## Security

Never commit credentials. Keep secrets in n8n's credential store or environment variables, and reference them with expressions such as `{{ $env.MY_API_KEY }}`.

## Contributing

Pull requests with new templates are welcome. Follow the layout above, strip all secrets, and describe the required credentials in the template's README.

## License

[MIT](LICENSE). Free to use, copy and adapt.
