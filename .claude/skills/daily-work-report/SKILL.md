---
name: daily-work-report
description: Generiše dnevni izveštaj o radu svih konsultanata i programera (po čoveku i po klijentu) iz SynapseD-a (Autotask + Jira) za zadati dan i snima ga kao Outlook draft. Bez argumenta podrazumeva JUČE. Koristi kada korisnik traži dnevni/izveštaj o radu, evidenciju sati po čoveku/klijentu, ili da se napravi/pošalje draft tog izveštaja.
---

# Dnevni izveštaj o radu (SynapseD)

Generiše dnevni izveštaj o radu svih konsultanata i programera iz Autotask-a i Jire (preko SynapseD MCP alata), grupisан **po čoveku** i **po klijentu**, i snima ga kao **Outlook draft** (ne šalje automatski).

## Argument
- Opcioni datum `YYYY-MM-DD`. Ako nije zadat → izveštaj za **juče** (današnji datum minus 1 dan).

## KRITIČNO PRAVILO
Svi sati — **i naplativi i nenaplativi** — svake osobe i svakog klijenta MORAJU biti u izveštaju. Nijedan sat se ne sme izgubiti.

## Poznati bug platforme (obavezno zaobići)
`timesheet_unified_get` i `timesheet_ingest` **tiho izbacuju** direktne Autotask time-entry-je na tiketima koji su „outsourcovani na Jiru" (`_outsourcedToJira`). Zato se **ne smeju** koristiti kao jedini izvor.
- Autoritativan izvor za KOMPLETNE sate je **`timesheet_weekly_company_report`** (čita entry-cache, sadrži sve unose).
- Nalog **`dev@uridium.rs`** (resourceID 29682885) su Jira worklog-ovi sinhronizovani nazad u Autotask (`internalNotes` = „Synced from Jira worklog … Logged by: <ime>") — NE brojati ih dvaput.

## Postupak
1. Odredi ciljni datum (argument ili juče). Format `YYYY-MM-DD`.
2. `mcp__Synapsed__timesheet_weekly_company_report(fromDate=dan, toDate=dan)` → autoritativni totali po osobi (email) i po klijentu, sa billable/non-billable. Iz njega izvuci i **listu svih osoba** koje su radile.
3. Za svaku osobu: `mcp__Synapsed__timesheet_replay(email, dan, dan)` (osveži cache), pa `mcp__Synapsed__timesheet_unified_get(email, dan, dan)` (opisi rada, tiketi, Jira issue-i). Zovi paralelno gde može.
4. **RECONCILE (obavezno):** za svaku osobu uporedi total iz koraka 2 sa zbirom iz koraka 3.
   - Ako je weekly total **veći** → postoje izbačeni unosi (outsourcovani tiketi). Pronađi koji (klijent, sati) nedostaju i dovuci taj tiket preko `mcp__Synapsed__autotask_workitem_get_by_number` (ili `autotask_workitem_get`); iz `timeEntries` izvuci unose tog `resourceID` za taj dan (sati + `summaryNotes` + `ticketNumber`/`ticketId`).
   - Nijedan sat iz weekly izveštaja ne sme ostati neprikazan.
5. **DE-DUPLIKACIJA:** unose pod `dev@uridium.rs` sa „Synced from Jira worklog … Logged by: <ime>" pripiši **stvarnom autoru** i ne prikazuj ih dodatno. Jira **Inbox** issue-e (projekat Inbox) pripiši stvarnom klijentu iz naslova `[Klijent]`.
6. Sastavi **HTML** na srpskom:
   - Zaglavlje: datum (DD.MM.YYYY.) + de-duplicirani zbir (ukupan rad / godišnji odmor / naplativo / nenaplativo).
   - Tabela **PO ČOVEKU** (sati + klijenti).
   - Tabela **PO KLIJENTU** (sati + ko je radio).
   - Detaljne stavke po čoveku: tiket + sati + klijent + kratak opis rada.
   - Sekcija **Odsutni** za `VacationTime` unose.
   - Ako opis nije mogao da se dovuče: prikaži sat + klijent + tiket uz napomenu „(opis na tiketu)".
7. **Linkovi** iza svakog broja tiketa/issue-a:
   - Autotask: `<a href="https://ww4.autotask.net/Mvc/ServiceDesk/TicketDetail.mvc?ticketID=TICKETID">TICKETNUMBER</a>`
   - Jira: `<a href="https://o-tri-o-n.atlassian.net/browse/ISSUEKEY">ISSUEKEY</a>`
   - Interne stavke (`CompanyTask`, `VacationTime`) bez linka.
8. `mcp__Synapsed__send_email`:
   - `to` = `dragana.krstec@uridium.rs, ivan.golubovic@uridium.rs, sasa.tabakovic@uridium.rs`
   - `subject` = `Dnevni izveštaj o radu — <DD.MM.YYYY.>`
   - `isHtml` = true, `body` = HTML.
   - Alat snima kao **draft** u Drafts folder — to je očekivano; ne forsiraj slanje.
9. Na kraju kratko izvesti: datum, de-duplicirani ukupan rad, broj ljudi, i da li je reconcile vratio izgubljene sate (i koliko).

## Ako nema podataka
Ako `weekly_company_report` za taj dan nema unosa (neradni dan), svejedno napravi draft sa napomenom da nema evidentiranog rada.

## Reference (vrednosti specifične za Uridium)
- Autotask zona: `ww4.autotask.net`, parametar linka: `ticketID`.
- Jira sajt: `o-tri-o-n.atlassian.net`, link: `/browse/<ISSUEKEY>`.
- Servisni nalog (duplikati Jira→Autotask): `dev@uridium.rs` / resourceID `29682885`.
- Podrazumevani primaoci: `dragana.krstec@uridium.rs, ivan.golubovic@uridium.rs, sasa.tabakovic@uridium.rs`.
