### 1.4 Example

```
### Group label
**Attribute** value
| Item | Attribute A | Attribute B |
|---|---|---|
| Row 1 | value 1 | value 2 |
| Row 2 | value 3 | value 4 |
```

## Markdown Parser
import ReactMarkdown from 'react-markdown';
import remarkGfm from 'remark-gfm';
import type { MarkdownBlock } from '../../../schemas/memo';
import './MarkdownText.css';

interface MarkdownTextProps {
  block: MarkdownBlock;
}

const HEADING = ({ children }: { children?: React.ReactNode }) => (
  <h3 className="md-text__heading">{children}</h3>
);

export function MarkdownText({ block }: MarkdownTextProps) {
  return (
    <div className="md-text">
      <ReactMarkdown
        remarkPlugins={[remarkGfm]}
        components={{
          h1: HEADING,
          h2: HEADING,
          h3: HEADING,
          h4: HEADING,
          h5: HEADING,
          h6: HEADING,
          p: ({ children }) => <p className="md-text__line">{children}</p>,
          table: ({ children }) => <table className="md-text__table">{children}</table>,
        }}
      >
        {block}
      </ReactMarkdown>
    </div>
  );
}

## Markdown CSS
.md-text {
  margin-top: var(--space-3);
}

.md-text__heading {
  font-size: var(--text-h3-size);
  font-weight: var(--text-h3-weight);
  color: var(--ubs-midnight-blue);
  margin-bottom: var(--space-3);
}

.md-text__line {
  font-size: 0.8125rem;
  line-height: 1.6;
  color: var(--ubs-charcoal);
}

.md-text__line + .md-text__line {
  margin-top: var(--space-2);
}

.md-text__line strong {
  font-weight: 600;
  color: var(--ubs-black);
}

.md-text__table {
  width: 100%;
  border-collapse: collapse;
  font-size: var(--text-caption-size);
  margin-top: var(--space-2);
}

.md-text__table + .md-text__table,
.md-text__line + .md-text__table,
.md-text__table + .md-text__line {
  margin-top: var(--space-3);
}

.md-text__table thead th {
  font-weight: 600;
  color: var(--ubs-gray-muted);
  padding: var(--space-1) var(--space-2);
  border-bottom: var(--border-hairline) solid var(--ubs-gray-border);
  text-align: left;
  white-space: nowrap;
}

.md-text__table tbody td {
  padding: var(--space-1) var(--space-2);
  border-bottom: var(--border-hairline) solid var(--ubs-gray-border);
  color: var(--ubs-charcoal);
  line-height: 1.4;
}

.md-text__table tbody tr:last-child td {
  border-bottom: none;
}

.md-text__table strong {
  font-weight: 600;
  color: var(--ubs-black);
}

.md-text ul,
.md-text ol {
  margin: var(--space-2) 0 0;
  padding-left: var(--space-5);
  font-size: 0.8125rem;
  line-height: 1.6;
  color: var(--ubs-charcoal);
}

.md-text li + li {
  margin-top: var(--space-1);
}

.md-text a {
  color: var(--ubs-midnight-blue);
  text-decoration: underline;
}

.md-text code {
  font-family: var(--font-mono);
  font-size: 0.75rem;
  background: var(--ubs-gray-faint);
  padding: 1px 4px;
  border-radius: var(--radius-sm);
}

### COMPAT TABLE

CompactTable

import type { TableBlock, TableColumn } from '../../../schemas/memo';
import './CompactTable.css';

interface CompactTableProps {
  block: TableBlock;
}

function colClass(col: TableColumn): string {
  switch (col.format) {
    case 'mono':    return 'mono';
    case 'number':  return 'mono right';
    case 'percent': return 'right';
    default:        return '';
  }
}

export function CompactTable({ block }: CompactTableProps) {
  const rows = block.rows.filter((r) => r.row_type === 'data');

  return (
    <div className="compact-table-wrap">
      {block.caption && <h3 className="compact-table__caption">{block.caption}</h3>}
      {block.subtitle && <p className="compact-table__subtitle">{block.subtitle}</p>}
      <table className="compact-table">
        <thead>
          <tr>
            {block.columns.map((col) => (
              <th key={col.key} className={colClass(col)}>
                {col.label}
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {rows.map((row, idx) => (
            <tr key={idx}>
              {block.columns.map((col) => (
                <td key={col.key} className={colClass(col)}>
                  {row.values[col.key]}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}


.compact-table-wrap {
  margin-top: var(--space-2);
  overflow-x: auto;
}

.compact-table__caption {
  font-size: var(--text-h3-size);
  font-weight: var(--text-h3-weight);
  color: var(--ubs-midnight-blue);
  margin-bottom: 2px;
}

.compact-table__subtitle {
  font-size: var(--text-caption-size);
  font-family: var(--font-mono, monospace);
  color: var(--ubs-gray-muted);
  margin-bottom: var(--space-2);
}

.compact-table {
  width: 100%;
  border-collapse: collapse;
  font-size: var(--text-caption-size);
}

.compact-table thead th {
  font-weight: 600;
  color: var(--ubs-gray-muted);
  padding: var(--space-1) var(--space-2);
  border-bottom: var(--border-hairline) solid var(--ubs-gray-border);
  text-align: left;
  white-space: nowrap;
}

.compact-table tbody td {
  padding: var(--space-1) var(--space-2);
  border-bottom: var(--border-hairline) solid var(--ubs-gray-border);
  color: var(--ubs-charcoal);
  line-height: 1.4;
}

.compact-table tbody tr:last-child td {
  border-bottom: none;
}

.compact-table .right {
  text-align: right;
}

