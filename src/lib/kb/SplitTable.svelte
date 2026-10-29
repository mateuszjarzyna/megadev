<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, splitCauseOf, uniqueById, type CauseGroup, type Report, type Tenant } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // `tickets`: cause → the issue it was split into, in `repo`.
  let { ticket, repo, tickets }: { ticket: number; repo: string; tickets: Partial<Record<CauseGroup['cause'], number>> } = $props();

  const rows = $derived((reports as Report[]).filter((r) => r.mergedInto === ticket));
  const merged = $derived(uniqueById(rows));
  const groups = $derived(
    groupByCause(merged, tenantOf, (r: Report, t: Tenant | undefined) => splitCauseOf(r, t, notes.oneDayMove)),
  );
  const tags = (g: CauseGroup) =>
    Object.entries(g.taggedAs)
      .sort((a, b) => b[1] - a[1])
      .map(([tag, n]) => `${tag} ×${n}`)
      .join(' · ');
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports in #{ticket} ({rows.length - merged.length} repeated rows in the export
  dropped) · <strong>{groups.length}</strong> {groups.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>cause</th><th>reports</th><th>tagged as by intake</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each groups as g (g.cause)}
      {@const n = tickets[g.cause]}
      <tr class:open={!n}>
        <td><code>{g.cause}</code></td>
        <td>{g.reports.length}</td>
        <td class="tags">{tags(g)}</td>
        <td>{g.tenants}</td>
        <td>{g.firstSeen} → {g.lastSeen}</td>
        <td>{#if n}<a href="https://github.com/{repo}/issues/{n}">{repo}#{n}</a>{:else}no ticket yet{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  .tags { color: var(--muted); font-size: 0.85em; }
  .open td { background: var(--warn-bg); }
  .open td:last-child { color: var(--warn); font-weight: 600; }
</style>
