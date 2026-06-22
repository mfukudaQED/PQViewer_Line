# PQViewer (Line)

English | [日本語](README.ja.md)


A lightweight browser-based 2D plotting viewer. It loads CSV/TSV/plain-text data and provides rich interactive visualization directly in your web browser.

---

## Features

- Drag & drop support for CSV / TSV / whitespace-delimited data
- Multi-series plotting (lines, points, error bars)
- Full axis control (range, log scale, tick formatting)
- Legend, label, and font customization
- Multiplot (subplot) layout
- PNG export
- Config export/import via JSON
- Gnuplot-like command interface
- Animation / frame playback support

---

## Usage

1. Open the web page
2. Drag and drop your data file (CSV/TSV/etc.)
3. Adjust settings using the Control Panel
4. Export the figure if needed

### Data Format

- Column 1: x values
- Column 2+: y series

---

## Project Structure

```
.
├── index.html   # Main application (single-file app)
├── README.md    # Documentation
└── LICENSE      # MPL-2.0 license
```

No build step is required. Everything runs in the browser.

---

## Dependencies

Loaded via CDN:

- MathJax (for LaTeX rendering)
- Google Fonts

No local installation is required.

---

## Notes

- Fully client-side implementation
- No data is uploaded externally
- Performance may degrade with very large datasets

---

## License

This project is licensed under the **Mozilla Public License 2.0 (MPL-2.0)**.

See the `LICENSE` file for the full license text.

Under MPL-2.0, modified source files covered by the license must remain available under MPL-2.0 when distributed. This README is not legal advice; consult the MPL-2.0 text for the exact terms.

---

## AI-Assisted Development

This viewer was developed with assistance from GPT-5.5 Think.
Parts of the source code were generated with AI assistance and were reviewed, modified, and maintained by the project author.

---

## Citation / acknowledgement

If this viewer helps prepare figures or analyze data for a publication,
please consider citing or acknowledging the repository according to the citation information provided by the project maintainers.


