
# README

To add new publications, add a new bib entry to `papers.bib`, and run `pandoc papers.bib -t csljson -o content/data/papers.json`. The `papers.json` file is automatically parsed by `layouts/shortcodes/publications.html`.

To add new student reports, add a new entry to `content/data/supervision.yml`. This is automatically parsed by `layouts/shortcodes/student_reports.html`.

To publish this website via Github pages, I used the following link: https://gohugo.io/host-and-deploy/host-on-github-pages/
