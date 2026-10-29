<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // `tickets`: cause → the issue it was split into, in `repo`. `shift` is the desk's one-day moves.
  let { ticket, repo, tickets }: { ticket: number; repo: string; tickets: Record<string, number> } = $props();

  const rows = $derived((reports as Report[]).filter((r) => r.mergedInto === ticket));
  const merged = $derived(uniqueById(rows));
  // Counted the desk's way (CauseTable): every report under the cause its tenant gives it.
  const groups = $derived(groupByCause(merged, tenantOf));
  // The one-day moves stay counted where they are filed, and get a row and a ticket of their own.
  const shifted = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const tags = (taggedAs: Record<string, number>) =>
    Object.entries(taggedAs)
      .sort((a, b) => b[1] - a[1])
      .map(([tag, n]) => `${tag} ×${n}`)
      .join(' · ');
  const link = (cause: string) => (tickets[cause] ? `${repo}#${tickets[cause]}` : null);
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports in #{ticket} ({rows.length - merged.length} repeated rows in the export
  dropped) · <strong>{groups.length + (shifted.length ? 1 : 0)}</strong> problems
</p>

<table>
  <thead>
    <tr><th>cause</th><th>reports</th><th>tagged as by intake</th><th>tenants</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each groups as g (g.cause)}
      {@const moved = readAs(g, notes.oneDayMove)}
      <tr>
        <td><code>{g.cause}</code></td>
        <td>{g.reports.length}{#if moved}<sup>*</sup>{/if}</td>
        <td class="tags">{tags(g.taggedAs)}</td>
        <td>{g.tenants}</td>
        <td>{#if link(g.cause)}<a href="https://github.com/{repo}/issues/{tickets[g.cause]}">{link(g.cause)}</a>{:else}no ticket{/if}</td>
      </tr>
    {/each}
    {#if shifted.length}
      <tr>
        <td><code>shift</code></td>
        <td>{shifted.length}<sup>*</sup></td>
        <td class="tags">{tags(shifted.reduce<Record<string, number>>((n, r) => ({ ...n, [r.meta.intakeTag]: (n[r.meta.intakeTag] ?? 0) + 1 }), {}))}</td>
        <td>{new Set(shifted.map((r) => r.tenant)).size}</td>
        <td>{#if link('shift')}<a href="https://github.com/{repo}/issues/{tickets.shift}">{link('shift')}</a>{:else}no ticket{/if}</td>
      </tr>
    {/if}
  </tbody>
</table>

{#each groups as g (g.cause)}
  {@const moved = readAs(g, notes.oneDayMove)}
  {#if moved}
    <p class="note">* {moved} of the {g.reports.length} <code>{g.cause}</code> reports are one-day moves: filed under
      <code>{g.cause}</code> by tenant, tracked as <code>shift</code>.</p>
  {/if}
{/each}

<style>
  .sum { font-size: 1.1rem; }
  .tags { color: var(--muted); font-size: 0.85em; }
  .note { color: var(--muted); font-size: 0.85em; }
</style>
