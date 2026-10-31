<!--

author:   Sebastian Zug & Claude Code (Opus 5.5)
email:    sebastian.zug@informatik.tu-freiberg.de
version:  0.1.0
language: en
narrator: US English Female
comment:  Interactive Karnaugh map for 2 to 4 variables: enter values, draw loops, read off DNF or CNF and compare with a minimal solution.

@Karnaugh.map
<lia-keep>
<karnaugh-map vars="@0" ones="@1" dc="@2"></karnaugh-map>
</lia-keep>
@end

@onload
if (!customElements.get('karnaugh-map')) {
  const COLORS = ['#0072B2', '#D55E00', '#009E73', '#CC79A7', '#E69F00', '#56B4E9']
  const CSS = `
    :host { display: block; margin: 1em 0; }
    .bar { display: flex; flex-wrap: wrap; gap: .4em; margin: .6em 0; }
    button { font: inherit; color: inherit; background: none; border: 1px solid currentColor; border-radius: 4px;
             padding: .25em .7em; cursor: pointer; }
    button:disabled { opacity: .4; cursor: default; }
    button.on { background: rgb(var(--color-highlight, 0, 106, 163)); border-color: transparent; color: #fff; }
    .grid { display: inline-grid; grid-auto-rows: 3em; }
    .head { display: flex; align-items: center; justify-content: center; padding: 0 .5em; font-size: 1.15em; }
    .cell { position: relative; border: 1px solid currentColor; border-radius: 0; margin: 0; padding: 0;
            font-size: 1.15em; font-family: monospace; }
    .cell.zero { opacity: .55; }
    .cell.dc { font-style: italic; background: rgba(128, 128, 128, .2); }
    .cell.sel { outline: 3px solid rgb(var(--color-highlight, 0, 106, 163)); outline-offset: -3px; font-weight: bold; }
    .cell small { position: absolute; right: 3px; bottom: 1px; font-size: .55em; opacity: .55; }
    .loop { pointer-events: none; border: 2.5px solid; border-radius: 12px; }
    .neg { display: inline-block; margin: 0 .1em; text-decoration: overline; }
    .row { margin: .5em 0; min-height: 1.5em; }
    .term { display: inline-block; border: 2px solid; border-radius: 12px; padding: 0 .5em; margin: .1em; }
    .term button { border: none; padding: 0 0 0 .4em; }
    .ok { font-weight: bold; }
  `
  const TEXT = {
    en: {
      edit: 'Enter values', loop: 'Draw loops', DNF: 'DNF (ones)', KNF: 'CNF (zeros)',
      add: 'Add loop', solve: 'Show solution', reset: 'Reset',
      hasWrong: v => `A loop must not contain a ${v}.`,
      badSize: c => `${c} cells: a loop covers 1, 2, 4, 8 or 16 cells.`,
      badShape: 'The cells do not form a rectangle, not even across the edge.',
      noTarget: v => `This loop covers no ${v}.`,
      hintEdit: 'A click cycles 0 → 1 → D.', hintLoop: 'Click the cells that form a loop.',
      grow: t => `Loop ${t} can be enlarged.`, spare: t => `Loop ${t} is redundant.`,
      left: (c, kind) => `Not covered yet: ${c} ${kind === 'DNF' ? (c === 1 ? 'one' : 'ones') : (c === 1 ? 'zero' : 'zeros')}.`,
      size: (l, c) => `${l} ${l === 1 ? 'loop' : 'loops'}, ${c} ${c === 1 ? 'literal' : 'literals'}`,
      notMinimal: (s, o) => `Correct, but not minimal: ${s}. Minimal would be ${o}.`,
      oneOf: c => `One of ${c} equally good solutions.`,
      cell: (m, v) => `Row ${m} of the truth table, value ${v}`, remove: i => `Remove loop ${i}`,
      invalid: 'A Karnaugh map needs 2 to 4 variables.',
    },
    de: {
      edit: 'Felder eintragen', loop: 'Schleifen bilden', DNF: 'DNF (Einsen)', KNF: 'KNF (Nullen)',
      add: 'Schleife übernehmen', solve: 'Lösung zeigen', reset: 'Zurücksetzen',
      hasWrong: v => `Eine Schleife darf keine ${v} enthalten.`,
      badSize: c => `${c} Felder: Eine Schleife umfasst 1, 2, 4, 8 oder 16 Felder.`,
      badShape: 'Die Felder bilden kein Rechteck, auch nicht über den Rand hinweg.',
      noTarget: v => `Diese Schleife deckt keine ${v} ab.`,
      hintEdit: 'Klick wechselt 0 → 1 → D.', hintLoop: 'Felder anklicken, die eine Schleife bilden.',
      grow: t => `Schleife ${t} lässt sich vergrößern.`, spare: t => `Schleife ${t} ist überflüssig.`,
      left: (c, kind) => `Noch nicht überdeckt: ${c} ${kind === 'DNF' ? (c === 1 ? 'Eins' : 'Einsen') : (c === 1 ? 'Null' : 'Nullen')}.`,
      size: (l, c) => `${l} ${l === 1 ? 'Schleife' : 'Schleifen'}, ${c} ${c === 1 ? 'Literal' : 'Literale'}`,
      notMinimal: (s, o) => `Korrekt, aber nicht minimal: ${s}. Minimal wären ${o}.`,
      oneOf: c => `Eine von ${c} gleich guten Lösungen.`,
      cell: (m, v) => `Zeile ${m} der Wahrheitstafel, Wert ${v}`, remove: i => `Schleife ${i} löschen`,
      invalid: 'Ein KV-Diagramm braucht 2 bis 4 Variablen.',
    },
  }
  const bits = x => { let c = 0; for (; x; x &= x - 1) c++; return c }
  const esc = x => x.replace(/[&<>"]/g, c => `&#${c.charCodeAt(0)};`)
  const name = v => esc(v).replace(/_(\w+)$/, '<sub>$1</sub>')
  const lit = (v, neg) => neg ? `<span class="neg">${name(v)}</span>` : name(v)

  // All blocks of 2^k cells: free = free variables, val = fixed values
  const cubes = n => {
    const list = [], all = (1 << n) - 1
    for (let free = 0; free <= all; free++)
      for (let val = 0; val <= all; val++) {
        if (val & free) continue
        let cells = 0
        for (let m = 0; m <= all; m++) if ((m & ~free) === val) cells |= 1 << m
        list.push({ free, val, cells })
      }
    return list
  }

  const primes = (n, target, dc) => {
    const ok = cubes(n).filter(c => !(c.cells & ~(target | dc)) && (c.cells & target))
    return ok.filter(c => !ok.some(d => d.cells !== c.cells && (d.cells & c.cells) === c.cells))
  }

  // All minimal covers: fewest loops first, then fewest literals
  const solve = (n, target, dc) => {
    const ps = primes(n, target, dc)
    const cost = sel => sel.length * 100 + sel.reduce((s, p) => s + n - bits(p.free), 0)
    let best = Infinity
    const found = new Map()
    const rec = (left, sel) => {
      if (sel.length * 100 > best) return
      if (!left) {
        const c = cost(sel)
        if (c > best) return
        if (c < best) { best = c; found.clear() }
        found.set(sel.map(p => p.cells).sort((a, b) => a - b).join(), sel)
        return
      }
      const bit = left & -left
      for (const p of ps) if (p.cells & bit) rec(left & ~p.cells, sel.concat(p))
    }
    rec(target, [])
    return { best: target ? best : 0, covers: [...found.values()] }
  }

  customElements.define('karnaugh-map', class extends HTMLElement {
    static solve(n, target, dc) { return solve(n, target, dc) }

    connectedCallback() {
      if (this.shadowRoot) return
      this.vars = this.getAttribute('vars').trim().split(/\s+/)
      const n = this.n = this.vars.length
      const lang = (this.closest('[lang]') || document.documentElement).lang || ''
      this.t = lang.toLowerCase().startsWith('de') ? TEXT.de : TEXT.en
      this.attachShadow({ mode: 'open' })
      if (n < 2 || n > 4) { this.shadowRoot.textContent = this.t.invalid; return }
      const parse = attr => (this.getAttribute(attr) || '').trim().split(/\s+/).filter(x => /^\d+$/.test(x))
        .map(Number).filter(x => x < 1 << n).reduce((m, x) => m | (1 << x), 0)
      const dc = parse('dc')
      this.start = { ones: parse('ones') & ~dc, dc }
      this.topBits = Math.ceil(n / 2)
      this.gray = k => k === 1 ? [0, 1] : [0, 1, 3, 2]
      this.reset()
    }

    reset() {
      this.ones = this.start.ones
      this.dc = this.start.dc
      this.mode = this.start.ones ? 'loop' : 'edit'
      this.kind = 'DNF'
      this.loops = []
      this.sel = 0
      this.note = ''
      this.render()
    }

    get size() { return 1 << this.n }
    get full() { return this.size === 32 ? -1 : (2 ** this.size) - 1 }
    get zeros() { return this.full & ~this.ones & ~this.dc }
    get target() { return this.kind === 'DNF' ? this.ones : this.zeros }

    // Checks the selection: returns the block, or why it is not a loop
    check(sel) {
      if (!sel) return { why: '' }
      if (sel & (this.full & ~(this.target | this.dc))) {
        return { why: this.t.hasWrong(this.kind === 'DNF' ? '0' : '1') }
      }
      const count = bits(sel)
      if (count & (count - 1)) return { why: this.t.badSize(count) }
      let and = this.size - 1, or = 0
      for (let m = 0; m < this.size; m++) if (sel & (1 << m)) { and &= m; or |= m }
      const free = and ^ or
      if (1 << bits(free) !== count) return { why: this.t.badShape }
      if (!(sel & this.target)) return { why: this.t.noTarget(this.kind === 'DNF' ? '1' : '0') }
      return { loop: { free, val: and, cells: sel } }
    }

    term(l) {
      const lits = this.vars.map((v, i) => {
        const b = 1 << (this.n - 1 - i)
        if (l.free & b) return null
        const set = !!(l.val & b)
        return lit(v, this.kind === 'DNF' ? !set : set)
      }).filter(Boolean)
      if (!lits.length) return this.kind === 'DNF' ? '1' : '0'
      return this.kind === 'DNF' ? lits.join('') : `(${lits.join(' + ')})`
    }

    click(m) {
      this.note = ''
      if (this.mode === 'edit') {
        const b = 1 << m
        if (this.ones & b) { this.ones &= ~b; this.dc |= b }
        else if (this.dc & b) this.dc &= ~b
        else this.ones |= b
        this.loops = this.loops.filter(l => this.check(l.cells).loop)
      } else this.sel ^= 1 << m
      this.render()
    }

    action(a) {
      this.note = ''
      if (a === 'edit' || a === 'loop') this.mode = a
      if (a === 'DNF' || a === 'KNF') { this.kind = a; this.loops = [] }
      if (a === 'edit' || a === 'DNF' || a === 'KNF') this.sel = 0
      if (a === 'add') {
        const r = this.check(this.sel)
        if (r.loop) { this.loops.push(r.loop); this.sel = 0 }
      }
      if (a === 'solve') {
        const { covers } = solve(this.n, this.target, this.dc)
        this.loops = (covers[0] || []).map(p => ({ ...p }))
        this.sel = 0
        this.mode = 'loop'
        this.note = covers.length > 1 ? this.t.oneOf(covers.length) : ''
      }
      if (a === 'reset') return this.reset()
      if (a.startsWith('del')) this.loops.splice(Number(a.slice(3)), 1)
      this.render()
    }

    // Position of a cell in the grid
    place(m) {
      const rowBits = this.n - this.topBits
      return {
        c: this.gray(this.topBits).indexOf(m >> rowBits),
        r: this.gray(rowBits).indexOf(m & ((1 << rowBits) - 1)),
      }
    }

    // Contiguous pieces: a loop across the edge splits into two or four
    pieces(l) {
      const seg = (set, len) => {
        const out = []
        for (let i = 0; i < len; i++) {
          if (!set.has(i)) continue
          if (out.length && out[out.length - 1].to === i - 1) out[out.length - 1].to = i
          else out.push({ from: i, to: i })
        }
        return out.map((s, i) => ({ ...s, openStart: out.length > 1 && i === 0, openEnd: out.length > 1 && i === out.length - 1 }))
      }
      const cs = new Set(), rs = new Set()
      for (let m = 0; m < this.size; m++) if (l.cells & (1 << m)) { const p = this.place(m); cs.add(p.c); rs.add(p.r) }
      const cols = seg(cs, 1 << this.topBits), rows = seg(rs, 1 << (this.n - this.topBits))
      return rows.flatMap(r => cols.map(c => ({ r, c })))
    }

    head(vars, code) {
      return vars.map((v, i) => lit(v, !(code & (1 << (vars.length - 1 - i))))).join('')
    }

    grid() {
      const top = this.vars.slice(0, this.topBits), side = this.vars.slice(this.topBits)
      const nc = 1 << this.topBits, nr = 1 << side.length
      let html = `<div class="grid" style="grid-template-columns: auto repeat(${nc}, 3.4em)">`
      this.gray(top.length).forEach((code, c) => {
        html += `<div class="head" style="grid-area: 1 / ${c + 2}">${this.head(top, code)}</div>`
      })
      this.gray(side.length).forEach((code, r) => {
        html += `<div class="head" style="grid-area: ${r + 2} / 1">${this.head(side, code)}</div>`
      })
      for (let m = 0; m < this.size; m++) {
        const { c, r } = this.place(m), b = 1 << m
        const v = this.ones & b ? '1' : this.dc & b ? 'D' : '0'
        const cls = ['cell', v === '0' ? 'zero' : v === 'D' ? 'dc' : '', this.sel & b ? 'sel' : ''].join(' ')
        html += `<button class="${cls}" data-m="${m}" style="grid-area: ${r + 2} / ${c + 2}"
                  aria-label="${this.t.cell(m, v)}" aria-pressed="${!!(this.sel & b)}">${v}<small>${m}</small></button>`
      }
      this.loops.forEach((l, i) => {
        const col = COLORS[i % COLORS.length], gap = 3 + (i % 4) * 3
        for (const { r, c } of this.pieces(l)) {
          const open = [r.openStart, c.openEnd, r.openEnd, c.openStart]
          const s = `grid-area: ${r.from + 2} / ${c.from + 2} / ${r.to + 3} / ${c.to + 3};
            margin: ${open.map(o => o ? 0 : gap + 'px').join(' ')};
            border-color: ${col}; background: ${col}22;
            border-radius: ${[open[0] || open[3], open[0] || open[1], open[2] || open[1], open[2] || open[3]].map(o => o ? 0 : '12px').join(' ')};
            ${open[0] ? 'border-top-style: none;' : ''} ${open[1] ? 'border-right-style: none;' : ''}
            ${open[2] ? 'border-bottom-style: none;' : ''} ${open[3] ? 'border-left-style: none;' : ''}`
          html += `<div class="loop" style="${s}"></div>`
        }
      })
      return html + '</div>'
    }

    // Feedback on the loops
    verdict() {
      const hints = []
      const ps = primes(this.n, this.target, this.dc)
      const union = cells => cells.reduce((u, c) => u | c, 0)
      this.loops.forEach((l, i) => {
        const others = union(this.loops.filter((_, j) => j !== i).map(o => o.cells))
        if (!ps.some(p => p.cells === l.cells)) hints.push(this.t.grow(this.term(l)))
        else if (!(l.cells & this.target & ~others)) hints.push(this.t.spare(this.term(l)))
      })
      const left = bits(this.target & ~union(this.loops.map(l => l.cells)))
      if (left) return `<div class="row">${this.t.left(left, this.kind)} ${hints.join(' ')}</div>`
      const { best, covers } = solve(this.n, this.target, this.dc)
      const cost = this.loops.length * 100 + this.loops.reduce((s, l) => s + this.n - bits(l.free), 0)
      const size = this.t.size(this.loops.length, cost % 100)
      if (cost === best) return `<div class="row ok">✓ Minimal: ${size}.</div>`
      const opt = this.t.size(Math.floor(best / 100), best % 100)
      return `<div class="row">${this.t.notMinimal(size, opt)} ${hints.join(' ')}</div>`
    }

    render() {
      const focus = this.shadowRoot.activeElement
      const key = focus && (focus.dataset.m !== undefined ? `[data-m="${focus.dataset.m}"]` : `[data-a="${focus.dataset.a}"]`)
      const tab = (a, label, on) => `<button data-a="${a}" class="${on ? 'on' : ''}" aria-pressed="${on}">${label}</button>`
      const res = this.check(this.sel)
      const terms = this.loops.map((l, i) => {
        return `<span class="term" style="border-color: ${COLORS[i % COLORS.length]}">${this.term(l)}<button data-a="del${i}"
                aria-label="${this.t.remove(i + 1)}">×</button></span>`
      }).join(this.kind === 'DNF' ? ' + ' : ' ')
      const none = this.target ? '' : (this.kind === 'DNF' ? '0' : '1')
      this.shadowRoot.innerHTML = `<style>${CSS}</style>
        <div class="bar">
          ${tab('edit', this.t.edit, this.mode === 'edit')}
          ${tab('loop', this.t.loop, this.mode === 'loop')}
          &nbsp;
          ${tab('DNF', this.t.DNF, this.kind === 'DNF')}
          ${tab('KNF', this.t.KNF, this.kind === 'KNF')}
        </div>
        ${this.grid()}
        <div class="bar">
          <button data-a="add" ${res.loop ? '' : 'disabled'}>${this.t.add}${res.loop ? ': ' + this.term(res.loop) : ''}</button>
          <button data-a="solve">${this.t.solve}</button>
          <button data-a="reset">${this.t.reset}</button>
        </div>
        <div aria-live="polite">
          <div class="row">${res.why || (this.mode === 'edit' ? this.t.hintEdit : this.sel ? '' : this.t.hintLoop)}</div>
          <div class="row"><i>f</i> = ${terms || none || '?'}</div>
          ${this.target ? this.verdict() : ''}
          ${this.note ? `<div class="row">${this.note}</div>` : ''}
        </div>`
      this.shadowRoot.querySelectorAll('[data-m]').forEach(b => { b.onclick = () => this.click(Number(b.dataset.m)) })
      this.shadowRoot.querySelectorAll('[data-a]').forEach(b => { b.onclick = () => this.action(b.dataset.a) })
      if (key) { const el = this.shadowRoot.querySelector(key); if (el) el.focus() }
    }
  })
}
@end

-->

# KarnaughMap - Template

An interactive Karnaugh map (Karnaugh-Veitch diagram) for 2 to 4 variables. Learners enter a Boolean function, draw loops and read off the result as a DNF or CNF. The widget explains why a selection is not a valid loop, and compares the result with a minimal solution. Don't-care cells are supported. It needs no external library.

Import the current version:

`import: https://raw.githubusercontent.com/LiaTemplates/KarnaughMap/master/README.md`

Or pin a version, which keeps a course stable:

`import: https://raw.githubusercontent.com/LiaTemplates/KarnaughMap/0.1.0/README.md`

The labels follow the language of the course: German for `language: de`, English otherwise.

## `@Karnaugh.map`

Call the macro on a line of its own:

```
@Karnaugh.map(variables,ones,don't-cares)
```

| Parameter     | Meaning                                                                    |
|---------------|----------------------------------------------------------------------------|
| `variables`   | 2 to 4 names separated by spaces, most significant first                   |
| `ones`        | truth table rows with the value 1, separated by spaces                     |
| `don't-cares` | truth table rows with the value D, separated by spaces                     |

Write an empty parameter as `-`. A trailing `_3` in a name becomes a subscript. With `ones` given, the map starts in the mode for drawing loops, otherwise in the mode for entering values. The first half of the variables labels the columns, the rest labels the rows.

A loop is selected cell by cell, so that loops across the edge work like any other loop.

## Example: four variables

@Karnaugh.map(w x y z,0 2 5 7 8 9 13 15,-)

## Example: don't-cares

Segment d of a seven-segment display: the inputs 10 to 15 do not occur.

@Karnaugh.map(A B C D,0 2 3 5 6 8 9,10 11 12 13 14 15)

## Example: subscripts

@Karnaugh.map(x_3 x_2 x_1,1 3 4 5,-)

## Example: empty map

Enter your own function.

@Karnaugh.map(x y z,-,-)

## Implementation

The macro wraps a web component in `<lia-keep>`, so it keeps its state across slide changes:

``` html
@Karnaugh.map
<lia-keep>
<karnaugh-map vars="@0" ones="@1" dc="@2"></karnaugh-map>
</lia-keep>
@end
```

The component `karnaugh-map` is defined in `@onload` and uses a shadow DOM. All implicants are enumerated, the prime implicants are filtered from them, and a search over the prime implicants finds all covers with the fewest loops, then the fewest literals. That is sufficient for 4 variables.
