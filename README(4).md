# Lab — GPO : Notifications & Blocage Windows Update sur Laptops

## Objectif

Mettre en place deux GPO sur un domaine Active Directory pour :
1. **Notifier** les utilisateurs de laptops lorsque des mises à jour Windows sont manquantes
2. **Bloquer** Windows Update sur ces mêmes laptops via le registre et le service

---

## Environnement

| Machine | Rôle | OS |
|---|---|---|
| WIN-UI39PMJB67B | Contrôleur de domaine | Windows Server 2019 |
| SEQUENCER | Laptop client | Windows 11 |
| Test | Laptop client | Windows 11 |

- **Domaine** : `sccm.lab`
- **OU cible** : `OU=Laptops,DC=sccm,DC=lab`
- **GPO** : `GPO-WU-Notification-Laptops`

---

## GPO 1 — Notification des mises à jour manquantes

### Étape 1 — Création de l'OU et du GPO

```powershell
Import-Module ActiveDirectory
Import-Module GroupPolicy

$domain = "sccm.lab"
$dc     = "DC=sccm,DC=lab"

# Créer l'OU Laptops
New-ADOrganizationalUnit -Name "Laptops" `
    -Path $dc `
    -ProtectedFromAccidentalDeletion $false

# Créer le GPO
New-GPO -Name "GPO-WU-Notification-Laptops" -Comment "Notif MAJ laptops"

# Lier le GPO à l'OU
New-GPLink -Name "GPO-WU-Notification-Laptops" `
    -Target "OU=Laptops,$dc" `
    -Enforced No
```

### Déplacer les laptops dans l'OU

```powershell
$dc = "DC=sccm,DC=lab"
$ouTarget = "OU=Laptops,$dc"
$laptops = @("SEQUENCER", "Test")

foreach ($name in $laptops) {
    $computer = Get-ADComputer -Identity $name
    Move-ADObject -Identity $computer.DistinguishedName `
                  -TargetPath $ouTarget
    Write-Host "Déplacé : $name" -ForegroundColor Cyan
}
```

### Étape 2 — Paramètres Windows Update via GPO

```powershell
$GPO  = "GPO-WU-Notification-Laptops"
$key  = "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate\AU"
$key2 = "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate"

Set-GPRegistryValue -Name $GPO -Key $key -ValueName "AUOptions" -Type DWord -Value 2
Set-GPRegistryValue -Name $GPO -Key $key -ValueName "NoAutoUpdate" -Type DWord -Value 0
Set-GPRegistryValue -Name $GPO -Key $key -ValueName "NoAutoRebootWithLoggedOnUsers" -Type DWord -Value 1
Set-GPRegistryValue -Name $GPO -Key $key2 -ValueName "SetUpdateNotificationLevel" -Type DWord -Value 1
Set-GPRegistryValue -Name $GPO -Key $key -ValueName "RebootRelaunchTimeout" -Type DWord -Value 5
Set-GPRegistryValue -Name $GPO -Key $key -ValueName "RebootWarningTimeout" -Type DWord -Value 30
```

| Paramètre | Valeur | Description |
|---|---|---|
| AUOptions | 2 | Notifie sans télécharger automatiquement |
| NoAutoUpdate | 0 | GPO actif |
| NoAutoRebootWithLoggedOnUsers | 1 | Pas de reboot forcé si user connecté |
| SetUpdateNotificationLevel | 1 | Toutes les notifications visibles |
| RebootRelaunchTimeout | 5 | Délai reboot en minutes |
| RebootWarningTimeout | 30 | Rappel reboot toutes les 30 min |

### Étape 3 — Script de notification

Déposé dans `\\sccm.lab\SYSVOL\sccm.lab\scripts\Check-Updates-Notify.ps1`

```powershell
$session  = New-Object -ComObject Microsoft.Update.Session
$searcher = $session.CreateUpdateSearcher()
$result   = $searcher.Search("IsInstalled=0 AND Type='Software'")
$count    = $result.Updates.Count

if ($count -gt 0) {
    $msg = "Ce poste a $count mise(s) a jour manquante(s). Merci de le mettre a jour."
    msg * $msg

    if (-not [System.Diagnostics.EventLog]::SourceExists("WU-Notify")) {
        New-EventLog -LogName Application -Source "WU-Notify" -ErrorAction SilentlyContinue
    }
    Write-EventLog -LogName Application -Source "WU-Notify" `
        -EventId 1001 -EntryType Warning `
        -Message "MAJ manquantes : $count" -ErrorAction SilentlyContinue
}
```

> **Pourquoi `msg.exe` ?** Les APIs WinRT Toast et le module BurntToast ont posé des problèmes de compatibilité sur Windows 11 (voir section Problèmes rencontrés). `msg.exe` est natif, sans dépendance, et fonctionne immédiatement.

### Étape 4 — Tâche planifiée

Créée directement sur les laptops via PowerShell :

```powershell
$script   = "\\sccm.lab\SYSVOL\sccm.lab\scripts\Check-Updates-Notify.ps1"
$trigger1 = New-ScheduledTaskTrigger -AtLogOn
$trigger2 = New-ScheduledTaskTrigger -RepetitionInterval (New-TimeSpan -Hours 2) -Once -At "00:00"
$action   = New-ScheduledTaskAction -Execute "powershell.exe" `
            -Argument "-ExecutionPolicy Bypass -NonInteractive -WindowStyle Hidden -File `"$script`""
$settings = New-ScheduledTaskSettingsSet -ExecutionTimeLimit (New-TimeSpan -Minutes 5) `
            -MultipleInstances IgnoreNew

Register-ScheduledTask -TaskName "WU-Check-Notify" `
    -Trigger $trigger1,$trigger2 `
    -Action $action `
    -Settings $settings `
    -RunLevel Limited -Force
```

La tâche s'exécute à chaque connexion utilisateur et toutes les 2 heures.

---

## GPO 2 — Blocage de Windows Update

### Étape 5 — Désactivation du service wuauserv (GUI gpmc.msc)

```
Éditeur GPO → GPO-WU-Notification-Laptops
→ Computer Configuration
→ Preferences
→ Control Panel Settings
→ Services
→ New → Service
```

| Champ | Valeur |
|---|---|
| Action | Remplacer |
| Nom du service | wuauserv |
| Type de démarrage | Désactivé |
| Action du service | Arrêter le service |

> **Limite** : Windows 11 protège ce service et le remet automatiquement en `Manuel`. C'est pourquoi on ajoute aussi le blocage par registre.

### Étape 6 — Blocage via registre (GUI gpmc.msc)

```
Éditeur GPO → GPO-WU-Notification-Laptops
→ Computer Configuration
→ Preferences
→ Windows Settings
→ Registry
→ New → Registry Item
```

Créer deux entrées :

| Ruche | Chemin | Valeur | Type | Données |
|---|---|---|---|---|
| HKLM | SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU | DisableWindowsUpdateAccess | REG_DWORD | 1 |
| HKLM | SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate | DisableWindowsUpdateAccess | REG_DWORD | 1 |

---

## Problèmes rencontrés

### 1. Here-string imbriqué dans PowerShell
Lors de la création du script dans SYSVOL, l'utilisation de `@"..."@` imbriqué dans un autre here-string provoquait l'erreur `UnexpectedCharactersAfterHereStringHeader`. Résolu en remplaçant les here-strings par des tableaux de strings `@(...)` et de la concaténation simple.

### 2. APIs WinRT Toast incompatibles sur Windows 11
Le chargement des assemblies `Windows.UI.Notifications` et `Windows.Data.Xml.Dom` via `ContentType=WindowsRuntime` échouait avec l'erreur `MissingAssemblyNameSpecification`. Plusieurs approches tentées sans succès. Abandonné au profit de `msg.exe`.

### 3. BurntToast introuvable sur les laptops
PSGallery n'était pas enregistré sur les laptops et l'accès Internet était limité. Testé via `Save-Module` sur le DC puis copie dans SYSVOL, mais l'installation sur les laptops a également échoué. Finalement abandonné car `msg.exe` était plus simple et fiable.

### 4. ExecutionPolicy bloquée sur les laptops
La politique d'exécution des scripts PowerShell était désactivée par défaut sur les laptops. Résolu en utilisant le paramètre `-ExecutionPolicy Bypass` dans les appels PowerShell sans modifier la politique globale.

### 5. Injection de la tâche planifiée via GPO Preferences
Le fichier `ScheduledTasks.xml` injecté dans le SYSVOL du GPO n'a pas été correctement appliqué sur les laptops à cause d'un problème de formatage XML lié aux here-strings imbriqués. La tâche a été créée directement sur les laptops avec `Register-ScheduledTask`.

### 6. Service wuauserv remis en Manuel par Windows 11
Windows 11 protège le service `wuauserv` et le remet automatiquement en `Manuel` même après désactivation via GPO Preferences. Contourné en ajoutant les clés de registre `DisableWindowsUpdateAccess`.

### 7. Laptop Test disparu de l'AD
Le laptop Test a perdu sa liaison avec le domaine (canal sécurisé cassé) et n'apparaissait plus dans l'AD malgré qu'il soit bien jointé à `sccm.lab`. Le GPO ne pouvait donc pas s'y appliquer. En cours de résolution via disjoint/rejoint au domaine.

---

## Vérification

```powershell
# GPO appliqué ?
gpresult /r /scope computer | findstr "GPO-WU"

# Tâche planifiée présente ?
Get-ScheduledTask -TaskName "WU-Check-Notify"

# Clés de registre en place ?
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
    -Name "DisableWindowsUpdateAccess"

# Logs de notification
Get-EventLog -LogName Application -Source "WU-Notify" -Newest 10

# Tester le script manuellement
powershell.exe -ExecutionPolicy Bypass -File "\\sccm.lab\SYSVOL\sccm.lab\scripts\Check-Updates-Notify.ps1"
```

---

## Résultat final

| Fonctionnalité | Méthode | Statut |
|---|---|---|
| Notification MAJ manquantes | Script `msg.exe` + tâche planifiée | ✅ Fonctionnel |
| Log des événements | Event Log Application (EventID 1001) | ✅ Fonctionnel |
| Désactivation service wuauserv | GPO Preferences → Services | ⚠️ Windows 11 remet en Manuel |
| Blocage accès Windows Update | GPO Preferences → Registre | ✅ Fonctionnel |
| Déploiement tâche via GPO Preferences | GPO Preferences → Scheduled Tasks | ⚠️ Créée manuellement sur les laptops |

---

## Commandes utiles

```powershell
# Forcer l'application du GPO
gpupdate /force

# Voir les machines dans l'OU
Get-ADComputer -Filter * -SearchBase "OU=Laptops,DC=sccm,DC=lab" | Select Name

# Voir le GPO lié à l'OU
Get-GPInheritance -Target "OU=Laptops,DC=sccm,DC=lab" | Select -Expand GpoLinks

# Voir tous les ordinateurs du domaine
Get-ADComputer -Filter * | Select Name, DistinguishedName
```
