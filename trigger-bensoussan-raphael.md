# EVAL 2 - Les triggers SQL de Raphaël Bensoussan

### 1. Trigger de Mise à Jour de la disponibilité des Voitures lors d'une Réservation

- **Nom du Trigger** : `maj_dispo_voiture`
- **Événement** : `AFTER INSERT` sur la table `Réservations`
- **Objectif** :
    - Il attend qu'une nouvelle réservation soit enregistrée.
    - Dès que c'est fait, il identifie la voiture concernée grâce à l'ID fourni dans la réservation.
    - Il met alors à jour la table "Voitures", changeant le statut de disponibilité de cette voiture spécifique à 0 (indisponible).
#### Code SQL :

```sql
DELIMITER //
CREATE TRIGGER maj_dispo_voiture
  AFTER INSERT
  ON Réservations
  FOR EACH ROW
BEGIN
  UPDATE Voitures
  SET disponible = 0
  WHERE id = NEW.voiture_id;
END;
//
```

### 2. Trigger de Vérification de l'Âge du Client avant une Réservation

- **Nom du Trigger** : `age_client_avant_reservation`
- **Événement** : `BEFORE INSERT` sur la table `Réservations`
- **Objectif** :
  - Il vérifie l'âge du client qui tente de faire la réservation.
  - Si le client a moins de 21 ans, il bloque la réservation et affiche un message d'erreur.

#### Code SQL :

```sql
DELIMITER //
CREATE TRIGGER age_client_avant_reservation
  BEFORE INSERT ON Réservations
  FOR EACH ROW
BEGIN
  IF (SELECT Clients.age FROM Clients WHERE id = NEW.client_id) < 21 THEN
        SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'Erreur: Lâge minimum pour louer une voiture est de 21 ans.';
END IF;
END;
//
```

### 3. Trigger de Validation du Permis de Conduire

- **Nom** : `verif_permis_conduire`
- **Événement** : `BEFORE INSERT` sur la table `Clients`
- **Objectif** :
  - Vérifier la validité et l'unicité du numéro de permis de conduire avant l'ajout d'un nouveau client.
  - Assurer que le permis a exactement 15 caractères alphanumériques.
  - Garantir l'unicité du numéro de permis dans la base de données.

#### Code SQL :

```sql
DELIMITER //

CREATE TRIGGER verif_permis_conduire
  BEFORE INSERT ON Clients
  FOR EACH ROW
BEGIN
  IF LENGTH(NEW.permis_conduire) != 15 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Le permis de conduire doit avoir exactement 15 caractères';
END IF;

IF NEW.permis_conduire NOT REGEXP '^[A-Z0-9]{15}$' THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Le permis de conduire doit être alphanumérique et avoir une longueur de 15 caractères';
END IF;

    IF EXISTS (SELECT 1 FROM Clients WHERE permis_conduire = NEW.permis_conduire) THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Le numéro de permis de conduire doit être unique';
END IF;
END;
//
```

### 4. Trigger de Vérification de la Disponibilité de la Voiture avant une Réservation

- **Nom du Trigger** : `dispo_voiture_reservation`
- **Événement** : `BEFORE INSERT` sur la table `Réservations`
- **Objectif** :
  - Il vérifie si la voiture que le client souhaite réserver est actuellement disponible.
  - Si la voiture n'est pas disponible, il empêche la réservation et affiche un message d'erreur.
  - Si la voiture est disponible, il autorise la réservation et affiche un message de confirmation.

#### Code SQL :

```sql
DELIMITER //
CREATE TRIGGER dispo_voiture_reservation
  BEFORE INSERT ON Réservations
  FOR EACH ROW
BEGIN
  IF (SELECT Voitures.disponible FROM Voitures WHERE id = NEW.voiture_id) = 0 THEN
        SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'Erreur: La voiture a déjà été réservée, elle nest actuellement pas disponible.';
  ELSE
    SIGNAL SQLSTATE '01000'
        SET MESSAGE_TEXT = 'La voiture est disponible pour réservation.';
END IF;
END;
//
```

### 5. Trigger pour Éviter les Chevauchements de Réservations sur la Même Voiture

- **Nom du Trigger** : `evite_reservation_meme_voiture`
- **Événement** : `BEFORE INSERT` sur la table `Réservations`
- **Objectif** :
  - Il vérifie si la voiture n'est pas déjà réservée pour la période demandée.
  - Si un chevauchement de dates est détecté, il bloque la réservation et affiche un message d'erreur.

#### Code SQL :

```sql
DELIMITER //

CREATE TRIGGER evite_reservation_meme_voiture
BEFORE INSERT ON Réservations
FOR EACH ROW
BEGIN
    IF (SELECT COUNT(*)
        FROM Réservations
        WHERE voiture_id = NEW.voiture_id
          AND (
              NEW.date_debut BETWEEN date_debut AND date_fin OR
              NEW.date_fin BETWEEN date_debut AND date_fin OR
              date_debut BETWEEN NEW.date_debut AND NEW.date_fin
          )
    ) > 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'La voiture est déjà réservée pour la période choisie.';
    END IF;
END;
//
```



