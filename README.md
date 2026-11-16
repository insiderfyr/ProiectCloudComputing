# ProiectCloudComputing: Social

Social e o aplicație cloud care ajută o companie să-și planifice și să-și publice postările pe Facebook și Instagram. Postările sunt scrise de un agent AI pe baza documentelor companiei. Fiecare afirmație trebuie să aibă o sursă, iar un om aprobă, editează sau respinge postarea înainte de publicare. Pe o postare aprobată nu se apasă niciodată „publică” fără ca cineva să fi văzut-o.

Proiectul pornește de la un modul existent din Amethys-Platform, care rulează acum local. Scopul pentru curs e să-l adaptez pentru Azure, cu cerințele de securitate și deploy care se cer.

## Cerințele cursului

- Servicii principale Azure: cel puțin o resursă de compute și cel puțin o bază de date sau object storage.
- Aplicația e securizată. Baza de date nu e expusă pe internet.
- Autentificare și autorizare, obligatorii.
- Coadă de mesaje, ideal, pentru comunicarea între servicii și pentru notificările email.
- Proiecte vechi sau de la alte cursuri pot fi folosite, dacă sunt adaptate pentru cloud.
- Aplicațiile mobile sunt opționale, cu backendul și baza de date în cloud.
- Cod cu funcționalități de bază, nu Hello World.
- Deploy scriptat (Azure CLI, ARM sau Terraform).
- Resursele se șterg după fiecare iterație, ca să nu se consume creditul.
- Plan de rezervă: Azure Free Account.

## Cum se îndeplinesc

| Cerință | Ce face proiectul | Cum se dovedește | Stare |
| --- | --- | --- | --- |
| Compute | Azure App Service (container Linux) cu integrare în VNet | URL-ul aplicației, `az webapp show` | Planificat |
| Bază de date | Azure Database for PostgreSQL Flexible Server, fără IP public | `az postgres flexible-server show`, firewall fără 0.0.0.0/0 | Planificat |
| Object storage | Azure Blob Storage pentru imagini și documente, container privat | Acces public dezactivat, acces doar prin identitate gestionată | Planificat |
| Securitate de rețea | Baza de date și Blob Storage sunt accesibile doar din VNet | Regulile NSG și configurația exportată | Planificat |
| Autentificare | Microsoft Entra ID, OpenID Connect | Login reușit în aplicație | Planificat |
| Autorizare | Roluri per client, verificate pe server | Teste pentru fiecare rol și test de izolare între doi clienți | Planificat |
| Coadă de mesaje | Azure Service Bus: `publish-jobs` și `notifications` | Mesaj trimis, procesat de worker | Planificat |
| Funcționalități de bază | Calendar, bibliotecă, aprobare, publicare programată, planificator | Teste de logică care trec | În lucru |
| Proiect vechi adaptat | Modulul Social din Amethys-Platform, mutat în Azure | Diferențele față de versiunea locală, documentate | În lucru |
| Mobil | Nu se folosește, interfața e web | Nu se aplică | Nu se aplică |
| Deploy scriptat | `scripts/deploy.sh` cu Azure CLI și Bicep | Deploy din zero într-un grup de resurse gol | Planificat |
| Teardown | `scripts/teardown.sh` șterge grupul de resurse | Lista de resurse e goală după rulare | Planificat |
| Plan de rezervă | Azure Free Account, dacă creditul se termină | Abonamentul activ, confirmat înainte de deploy | Planificat |

## Arhitectura

Utilizatorul intră în aplicație prin HTTPS și se autentifică cu Entra ID. API-ul, rulat pe App Service, verifică tokenul și rolul, apoi lucrează cu PostgreSQL și Blob Storage, amândouă accesibile doar din VNet.

Publicarea nu se face direct din API. Când vine ora unei postări aprobate, API-ul pune un job în coada `publish-jobs`. Un worker separat consumă jobul, apelează API-ul Meta și scrie rezultatul. Jobul are o cheie unică, astfel încât o postare nu poate apărea de două ori. Dacă răspunsul Meta nu ajunge sau nu e clar, postarea trece în starea „necunoscută”, iar decizia rămâne la om. Sistemul nu reîncearcă singur.

Notificările pentru aprobare, respingere sau eșec merg prin coada `notifications` și sunt trimise pe email de un consumator. Un mesaj care nu s-a trimis rămâne în coadă.

Secretele sunt în Key Vault, inclusiv tokenul Meta, criptat. Aplicația și workerul folosesc identitate gestionată, deci în cod nu există parole. Repo-ul conține doar `.env.example`, cu numele variabilelor.

| Componentă | Serviciu | Acces |
| --- | --- | --- |
| API și interfața web | App Service | HTTPS public, restul intern |
| Worker de publicare | Container Apps sau al doilea App Service | Doar din VNet și Service Bus |
| Date | PostgreSQL Flexible Server | Doar din VNet |
| Fișiere | Blob Storage | Doar prin identitate gestionată |
| Mesaje | Service Bus | Identitate gestionată |
| Secrete | Key Vault | Identitate gestionată |
| Jurnale și alerte | Application Insights, Log Analytics | Intern |

## Autentificare și autorizare

Clienții sunt separați prin `tenant_id`, prezent în fiecare tabel. Fiecare utilizator are un rol într-un client:

- **Owner**: administrează clientul, utilizatorii și canalele.
- **Editor**: scrie și modifică postări.
- **Approver**: aprobă, editează ora sau respinge.
- **Viewer**: vede calendarul, fără acțiuni.

Rolul se verifică pe server, la fiecare cerere. Fără token valid răspunsul e `401`, iar cu rol insuficient e `403`. Un test automat încearcă accesul unui utilizator din clientul A la date din clientul B și trebuie să primească `403` sau `404`.

## Fluxuri

**Generarea.** Agentul citește documentele clientului și temele lui. Redactează postarea, apoi verifică în cod că fiecare cifră și fiecare afirmație are un citat exact într-o sursă, că lungimea și hashtag-urile respectă canalul și că nu repetă o postare anterioară. Ce nu poate fi susținut ajunge la om, cu explicația.

**Publicarea.** Postarea trece prin trei stări posibile la publicare: publicată (se salvează ID-ul Meta), eșuată (omul poate reîncerca) sau necunoscută (nu se reîncearcă automat). Starea e vizibilă în calendar, cu acțiunile Anulează și Reîncearcă.

## Model de date

- `tenant`: nume, buget lunar, plafon de postări pe zi și pe canal
- `user_membership`: utilizator, client, rol
- `channel`: platformă, ID-ul paginii Meta, token criptat
- `post`: canal, text, surse, ora programată, stare, `idempotency_key`
- `approval`: postare, decizie, utilizator, moment
- `audit_event`: client, actor, acțiune, obiect, moment

## Deploy și iterații

Lucrul se desfășoară în iterații. La început se rulează `deploy.sh`, care creează grupul de resurse, rețeaua, serviciile și aplicația din parametrii din `infra/`. Scriptul poate fi rulat din nou fără să dubleze resursele. La final, `teardown.sh` șterge grupul de resurse. Între iterații nu rămâne nimic pornit.

Costurile sunt ținute mici prin SKU-uri de dezvoltare (App Service B1, PostgreSQL Burstable, Service Bus Basic) și printr-un buget lunar cu alertă la 80%.

## Stadiu

Logica și ecranele modulului Social există și sunt testate în proiectul original. Pentru Azure urmează adaptarea configurării (variabile de mediu, Key Vault, identitate gestionată), scripturile Bicep și de deploy, rețeaua privată, coada, worker-ul și autentificarea Entra ID.

Pentru lansare mai rămân câteva lucruri care depind de terți: o pagină Facebook și un cont Instagram de test, un model AI real, suita de teste de browser, avizul juridic, monitorizarea tokenului Meta și migrările pe copia producției.

## Structura repo-ului

```
app/        API și interfața web
worker/     consumatorul cozii de publicare
infra/      Bicep și parametri
scripts/    deploy.sh, teardown.sh
tests/      teste de logică, autorizare și integrare
```

## Rulare

Instrucțiunile complete apar când scripturile există. Fluxul planificat e:

```bash
./scripts/deploy.sh --env dev
# lucru și teste
./scripts/teardown.sh --env dev
```
