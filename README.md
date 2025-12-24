Use this code to scrape data from MOE school finder:
(async () => {

  const YEAR = 2021;
  const sleep = ms => new Promise(r => setTimeout(r, ms));
  const results = [];

  function clean(text) {
    return text ? text.replace(/\s+/g, ' ').trim() : '';
  }

  function splitRange(text) {
    if (!text || text === '-') return { min: '', max: '' };
    const m = text.match(/(\d+)\s*-\s*(\d+)/);
    return m ? { min: m[1], max: m[2] } : { min: '', max: '' };
  }

  function emptyRow(school, tableIndex, stream) {
    return {
      school,
      table_index: tableIndex,
      year: YEAR,
      stream,
      ip_min: '',
      ip_max: '',
      pg3_min: '',
      pg3_max: '',
      pg2_min: '',
      pg2_max: '',
      pg1_min: '',
      pg1_max: ''
    };
  }

  function extractTables(doc, schoolName) {
    const tables = [...doc.querySelectorAll('table.moe-table')];

    tables.forEach((table, idx) => {
      const tableIndex = idx + 1;

      const affiliatedRow = emptyRow(schoolName, tableIndex, 'Affiliated');
      const nonAffRow   = emptyRow(schoolName, tableIndex, 'Non-affiliated');

      table.querySelectorAll('tbody tr').forEach(row => {
        const programme = clean(row.querySelector('th')?.innerText);
        const cells = row.querySelectorAll('td p');
        if (cells.length < 2) return;

        const aff = splitRange(clean(cells[0].innerText));
        const nonAff = splitRange(clean(cells[1].innerText));

        if (/Integrated Programme/i.test(programme)) {
          affiliatedRow.ip_min = aff.min;
          affiliatedRow.ip_max = aff.max;
          nonAffRow.ip_min = nonAff.min;
          nonAffRow.ip_max = nonAff.max;
        }

        if (/Posting Group 3/i.test(programme)) {
          affiliatedRow.pg3_min = aff.min;
          affiliatedRow.pg3_max = aff.max;
          nonAffRow.pg3_min = nonAff.min;
          nonAffRow.pg3_max = nonAff.max;
        }

        if (/Posting Group 2/i.test(programme)) {
          affiliatedRow.pg2_min = aff.min;
          affiliatedRow.pg2_max = aff.max;
          nonAffRow.pg2_min = nonAff.min;
          nonAffRow.pg2_max = nonAff.max;
        }

        if (/Posting Group 1/i.test(programme)) {
          affiliatedRow.pg1_min = aff.min;
          affiliatedRow.pg1_max = aff.max;
          nonAffRow.pg1_min = nonAff.min;
          nonAffRow.pg1_max = nonAff.max;
        }
      });

      results.push(affiliatedRow, nonAffRow);
    });
  }

  async function scrapeSchool(url) {
    try {
      const res = await fetch(url);
      const html = await res.text();
      const doc = new DOMParser().parseFromString(html, 'text/html');

      const schoolName = clean(
        doc.querySelector('h1.m-b\\:l')?.innerText
      );

      if (!schoolName) {
        console.warn('⚠ School name missing:', url);
        return;
      }

      extractTables(doc, schoolName);
      console.log(`✔ Scraped: ${schoolName}`);
    } catch (e) {
      console.warn(`✖ Failed: ${url}`, e);
    }
  }

  async function scrapePage() {
    const links = [...document.querySelectorAll(
      'a[href*="schooldetail?schoolname="]'
    )];

    for (const link of links) {
      await scrapeSchool(link.href);
      await sleep(500);
    }
  }

  async function nextPage() {
    const btn = document.querySelector(
      'button.moe-pagination__btn.dir--right:not([disabled])'
    );
    if (!btn) return false;
    btn.click();
    await sleep(2500);
    return true;
  }

  // ▶ MAIN LOOP
  let page = 1;
  while (true) {
    console.log(`📄 Page ${page}`);
    await scrapePage();
    if (!(await nextPage())) break;
    page++;
  }

  // ▶ CSV EXPORT (TypeError FIXED)
  const headers = [
    'school','table_index','year','stream',
    'ip_min','ip_max',
    'pg3_min','pg3_max',
    'pg2_min','pg2_max',
    'pg1_min','pg1_max'
  ];

  const csv = [
    headers.join(','),
    ...results.map(r =>
      headers.map(h =>
        `"${String(r[h] ?? '').replace(/"/g, '""')}"`
      ).join(',')
    )
  ].join('\n');

  const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = 'secondary_school_cutoff_points_normalised.csv';
  a.click();

  console.log(`✅ Done — ${results.length} rows exported`);
})();
