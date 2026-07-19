### 2.1 Problem statement

A flat "copy everything" action doesn't scale once content is organized into sections, subsections, and — for some
subsections — multiple scoped variants of the same table (e.g., the same set of metrics shown once per scope). The user
needs to choose *which* of those to include, and that choice set needs to be driven by configuration, not hardcoded
per-button logic, so what's copyable can change without touching the selector component itself.

### 2.2 Content-tree shape assumed

This spec assumes a tree of:

```
Section
  └── Subsection (zero or more)
        └── Content blocks (one or more), each with a content_format
```

Some subsections may hold a **scoped/repeated content format** — a content format where a single subsection's content is
actually an array of otherwise-identical blocks, each tagged with a different "scope" (e.g. an aggregate view vs. a
component-level view of the same metrics). This spec calls that format `SCOPED_TABLE`; the scope tag type is a small
closed enum specific to your domain (illustrative example below uses `'SCOPE_A' | 'SCOPE_B' | 'SCOPE_C'` — replace with
your real scope taxonomy):

### 2.4 Selector eligibility rules

An entry appears in the selector if and only if:

1. `visibleOnEnvironment` includes the current environment, **and**
2. It has an entry in `actions` for the relevant action name with `enabled: true`

Eligible entries render in `displayOrder` order. A checkbox's initial checked state is that action's `defaultValue`.

### 2.5 UI structure

Three possible levels of nesting, rendered as a tree of checkboxes:

```
[ ] Section A
      [ ] Subsection A.1
      [ ] Subsection A.2
            Scope
            [ ] Scope 1
            [ ] Scope 2
            [ ] Scope 3
[ ] Section B          (no subsections → just the one checkbox)
```

Rules:

- A subsection checkbox is **disabled** (visible, greyed out, not hidden) when its parent section is unchecked.
- A scope checkbox is disabled when either its parent subsection or that subsection's parent section is unchecked.
- Disabling a control does not clear its checked state — re-enabling a parent restores whatever the child was previously
  set to.
- A section with zero eligible subsections renders as a single checkbox with no nested group.
- The scope group only renders under a subsection whose config entry has a `contentFilter` (i.e., its content format is
  the scoped one).

**Presentation — prefer a centered modal over an anchored/positioned popover once nesting is involved.** A popover
anchored to the triggering button is fine for a flat, single-level list, but two extra nesting levels (subsection,
scope) need room to grow that a small anchored box doesn't have. A centered modal — dimmed backdrop, fixed header and
footer, scrollable middle region — scales to however deep and wide the configured tree gets. If your use case is
guaranteed to stay flat (no subsections, no scoped content), the lighter-weight anchored popover is the better default;
don't build the modal's extra scaffolding speculatively.

### 2.6 Text-serialization rules

Given a live selection state (which section/subsection checkboxes are checked, and which scope checkboxes are checked
per scoped subsection), produce plain text as follows:

1. If the section's config entry has `showHeading: true` (or has no entry — defaults to `true`), emit a top-level
   heading with the section's name.
2. For each subsection, **in order**:
    - If it's unchecked in the current selection, skip it entirely — including its heading. It contributes nothing to
      the output.
    - Otherwise, if the section's `showSubHeading` is `true` **and** the subsection's own `showHeading` is `true` (or
      missing), emit a sub-heading with the subsection's title.
    - Serialize each of its content blocks per its content format (table below).
3. If a section has no subsections, serialize its own content blocks directly, after the section heading.
4. Join all non-empty parts with a blank line between them.

Per-content-format plain-text rendering — this is the complete contract, not just the two formats introduced here:

| Content format                                            | Text rendering                                                                                                                                                                                                                                                                                 |
|-----------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `MD`                                                      | The raw string, verbatim                                                                                                                                                                                                                                                                       |
| List-shaped                                               | Heading line, then one bullet per item                                                                                                                                                                                                                                                         |
| Key/value-shaped                                          | Heading line, then `label: value` per row                                                                                                                                                                                                                                                      |
| Tagged-list-shaped (e.g. dated items with a category tag) | Heading line, then `[date] tag: text` per item                                                                                                                                                                                                                                                 |
| Table-shaped                                              | Caption, subtitle, pipe-joined header row, pipe-joined data rows                                                                                                                                                                                                                               |
| `SCOPED_TABLE` (§2.2)                                     | One block per **visible** scope only: `[SCOPE]`, caption, pipe-joined header, pipe-joined rows. A scope is visible if the live selection's scope-checkbox state says so; if no live selection was passed for that subsection, fall back to the config's `contentFilter.scopes[scope].visible`. |
| Alert/notice-shaped (severity + title + optional body)    | `[SEVERITY] title — body`                                                                                                                                                                                                                                                                      |
| Chart-shaped, loading/skeleton placeholder                | Not serialized — no sensible plain-text form, contributes nothing       


## POPUP 
import { useEffect, useMemo, useState } from 'react';
import type { CreditSummarySection, CreditSummarySubSection, MatrixScope } from '../../schemas/memo';
import { Button, X } from '../../primitives';
import { useUi } from '../../hooks/useUi';
import { MOCK_MEMO } from '../../mocks/memo-fixtures';
import { SECTION_ACTIONS_CONFIG, CURRENT_ENVIRONMENT, type SectionActionConfig } from '../../config/sectionActionsConfig';
import { serializeSectionToText, type ScopeOverrides } from './serializeSection';
import { MATRIX_SCOPE_LABEL, MATRIX_SCOPES } from './matrixScopeLabels';
import './CopyToClipboardPopover.css';

interface CopyToClipboardPopoverProps {
  isOpen: boolean;
  onClose: () => void;
}

interface EligibleSubsection {
  sub: CreditSummarySubSection;
  cfg: SectionActionConfig;
}

interface EligibleSection {
  id: string;
  section: CreditSummarySection;
  defaultValue: boolean;
  subsections: EligibleSubsection[];
}

function getConfig(id: string) {
  return SECTION_ACTIONS_CONFIG.find((c) => c.id === id);
}

function isEnabledHere(cfg: SectionActionConfig | undefined): cfg is SectionActionConfig {
  return !!cfg
    && cfg.visibleOnEnvironment.includes(CURRENT_ENVIRONMENT)
    && !!cfg.actions.find((a) => a.action === 'copy to clipboard' && a.enabled);
}

function defaultOf(cfg: SectionActionConfig): boolean {
  return cfg.actions.find((a) => a.action === 'copy to clipboard')!.defaultValue;
}

function getEligibleSections(): EligibleSection[] {
  return SECTION_ACTIONS_CONFIG
    .filter(isEnabledHere)
    .map((cfg) => ({ cfg, section: MOCK_MEMO.sections.find((s) => s.section_id === cfg.id) }))
    .filter((e): e is { cfg: SectionActionConfig; section: CreditSummarySection } => !!e.section)
    .sort((a, b) => a.cfg.displayOrder - b.cfg.displayOrder)
    .map(({ cfg, section }) => {
      const subsections = (section.subsections ?? [])
        .map((sub) => ({ sub, cfg: getConfig(sub.subsection_id) }))
        .filter((e): e is EligibleSubsection => isEnabledHere(e.cfg))
        .sort((a, b) => a.cfg.displayOrder - b.cfg.displayOrder);
      return { id: cfg.id, section, defaultValue: defaultOf(cfg), subsections };
    });
}

export function CopyToClipboardPopover({ isOpen, onClose }: CopyToClipboardPopoverProps) {
  const { showToast } = useUi();
  const eligible = useMemo(() => getEligibleSections(), []);
  const [checked, setChecked] = useState<Record<string, boolean>>({});
  const [scopeChecked, setScopeChecked] = useState<ScopeOverrides>({});

  useEffect(() => {
    if (!isOpen) return;
    const checkedState: Record<string, boolean> = {};
    const scopeState: ScopeOverrides = {};
    for (const e of eligible) {
      checkedState[e.id] = e.defaultValue;
      for (const { sub, cfg } of e.subsections) {
        checkedState[sub.subsection_id] = defaultOf(cfg);
        if (cfg.contentFilter) {
          scopeState[sub.subsection_id] = Object.fromEntries(
            MATRIX_SCOPES.map((scope) => [scope, cfg.contentFilter!.scopes[scope]?.visible ?? false]),
          ) as Record<MatrixScope, boolean>;
        }
      }
    }
    setChecked(checkedState);
    setScopeChecked(scopeState);
  }, [isOpen, eligible]);

  useEffect(() => {
    if (!isOpen) return;
    const handleKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    window.addEventListener('keydown', handleKey);
    return () => window.removeEventListener('keydown', handleKey);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  const toggle = (id: string) => setChecked((c) => ({ ...c, [id]: !c[id] }));
  const toggleScope = (subsectionId: string, scope: MatrixScope) =>
    setScopeChecked((s) => ({
      ...s,
      [subsectionId]: { ...s[subsectionId], [scope]: !s[subsectionId]?.[scope] },
    }));

  const selected = eligible.filter((e) => checked[e.id]);

  const handleCopy = async () => {
    const text = selected
      .map((e) => serializeSectionToText(e.section, { scopeOverrides: scopeChecked, subsectionSelection: checked }))
      .join('\n\n---\n\n');
    try {
      await navigator.clipboard.writeText(text);
      showToast(`Copied ${selected.length} section${selected.length !== 1 ? 's' : ''} to clipboard`);
    } catch {
      showToast('Failed to copy');
    }
    onClose();
  };

  return (
    <>
      <div className="copy-popover-scrim" onClick={onClose} aria-hidden="true" />
      <div className="copy-popover" role="dialog" aria-label="Select sections to copy">
        <div className="copy-popover__header">
          <span>Select sections to copy</span>
          <button className="copy-popover__close" onClick={onClose} aria-label="Close">
            <X size={16} />
          </button>
        </div>
        <div className="copy-popover__list">
          {eligible.map(({ id, section, subsections }) => (
            <div key={id}>
              <label className="copy-popover__item">
                <input type="checkbox" checked={!!checked[id]} onChange={() => toggle(id)} />
                {section.section_name}
              </label>
              {subsections.length > 0 && (
                <div className="copy-popover__subsection-group">
                  {subsections.map(({ sub, cfg }) => (
                    <div key={sub.subsection_id}>
                      <label className="copy-popover__subitem">
                        <input
                          type="checkbox"
                          disabled={!checked[id]}
                          checked={!!checked[sub.subsection_id]}
                          onChange={() => toggle(sub.subsection_id)}
                        />
                        {sub.title}
                      </label>
                      {cfg.contentFilter && (
                        <div className="copy-popover__scope-group">
                          <span className="copy-popover__scope-label">Scope</span>
                          {MATRIX_SCOPES.map((scope) => (
                            <label key={scope} className="copy-popover__scope-item">
                              <input
                                type="checkbox"
                                disabled={!checked[id] || !checked[sub.subsection_id]}
                                checked={!!scopeChecked[sub.subsection_id]?.[scope]}
                                onChange={() => toggleScope(sub.subsection_id, scope)}
                              />
                              {MATRIX_SCOPE_LABEL[scope]}
                            </label>
                          ))}
                        </div>
                      )}
                    </div>
                  ))}
                </div>
              )}
            </div>
          ))}
        </div>
        <div className="copy-popover__footer">
          <Button variant="ghost" onClick={onClose}>Cancel</Button>
          <Button variant="primary" onClick={handleCopy} disabled={selected.length === 0}>
            Copy {selected.length > 0 ? `(${selected.length})` : ''}
          </Button>
        </div>
      </div>
    </>
  );
}


.copy-popover-scrim {
  position: fixed;
  inset: 0;
  z-index: 900;
  background: rgba(0, 0, 0, 0.35);
}

.copy-popover {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 910;
  width: 420px;
  max-width: calc(100vw - var(--space-5) * 2);
  max-height: calc(100vh - var(--space-6) * 2);
  background: var(--ubs-pure-white);
  border-radius: var(--radius-sm);
  box-shadow: 0 16px 40px rgba(0, 0, 0, 0.18);
  display: flex;
  flex-direction: column;
}

.copy-popover__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: var(--space-4) var(--space-5);
  border-bottom: var(--border-hairline) solid var(--ubs-gray-border);
  font-size: 1rem;
  font-weight: 700;
  color: var(--ubs-black);
  flex-shrink: 0;
}

.copy-popover__close {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: var(--radius-sm);
  color: var(--ubs-gray-muted);
}

.copy-popover__close:hover {
  background: var(--ubs-gray-faint);
  color: var(--ubs-charcoal);
}

.copy-popover__list {
  display: flex;
  flex-direction: column;
  gap: var(--space-4);
  padding: var(--space-4) var(--space-5);
  overflow-y: auto;
}

.copy-popover__item {
  display: flex;
  align-items: center;
  gap: var(--space-2);
  font-size: var(--text-body-size);
  font-weight: 600;
  color: var(--ubs-black);
  cursor: pointer;
}

.copy-popover__subsection-group {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  margin: var(--space-2) 0 0 var(--space-5);
  padding-left: var(--space-3);
  border-left: var(--border-hairline) solid var(--ubs-gray-border);
}

.copy-popover__subitem {
  display: flex;
  align-items: center;
  gap: var(--space-2);
  font-size: var(--text-body-size);
  color: var(--ubs-charcoal);
  cursor: pointer;
}

.copy-popover__subitem:has(input:disabled) {
  color: var(--ubs-gray-muted);
  cursor: not-allowed;
}

.copy-popover__scope-group {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
  margin: var(--space-1) 0 0 var(--space-5);
  padding-left: var(--space-2);
  border-left: var(--border-hairline) solid var(--ubs-gray-border);
}

.copy-popover__scope-label {
  font-size: var(--text-caption-size);
  color: var(--ubs-gray-muted);
  margin-bottom: 2px;
}

.copy-popover__scope-item {
  display: flex;
  align-items: center;
  gap: var(--space-2);
  font-size: var(--text-caption-size);
  color: var(--ubs-charcoal);
  cursor: pointer;
}

.copy-popover__scope-item:has(input:disabled) {
  color: var(--ubs-gray-muted);
  cursor: not-allowed;
}

.copy-popover__footer {
  display: flex;
  justify-content: flex-end;
  gap: var(--space-2);
  padding: var(--space-4) var(--space-5);
  border-top: var(--border-hairline) solid var(--ubs-gray-border);
  flex-shrink: 0;
}
|

The live selection state should always be able to **override** the config's static default scope visibility — the config
only sets what's pre-checked when the selector opens, not a fixed rule enforced afterward.
