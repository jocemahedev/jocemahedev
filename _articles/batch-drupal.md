---
layout: single
title: "Formulaire import Batch"
---

Objectif:

Lancer des opérations en Batch sur les lignes d’un fichier excel,csv.

- Formulaire du fichier d’upload

On créé d’abord le formulaire d’upload, avec une restriction sur le type de fichier a uploader

```php
	/**
   * {@inheritdoc}.
   */
  public function buildForm(array $form, FormStateInterface $form_state) {
    // Construction du formulaire d'upload.
    $form['file'] = [
      '#type' => 'file',
      '#title' => "Fournir un fichier ",
      '#size' => 20,
      '#prefix' => "<div class='file-form-import'>",
      '#suffix' => "</div>",
      '#upload_validators'  => [
    // On restreint les fichiers aux extensions xlsx et csv.
        'file_validate_extensions' => ['xlsx', 'csv'],
      ],
      '#description' => 'Fichier valide : xlsx,csv avec feuille nommée Import',
    ];

    $form['actions']['#type'] = 'actions';
    $form['actions']['submit'] = [
      '#type' => 'submit',
      '#value' => $this->t('Lancer le traitement du fichier'),
    ];

    return $form;
  }
```

- Fonction du traitement du fichier

On créé une fonction qui permet de traiter le fichier uploader

Au préalable il faut installer les librairies utiles via composer

```php
composer require phpoffice/phpspreadsheet
```

On fait au plus simple ici pour la fonction on boucle sur la première colonne de chaque ligne du fichier.

```php
/**
   * Traitement du fichier uploadé
   * ici on itère juste sur la première colonne à partir de la ligne 1
   */
  public function import($file_path, &$context) {
    $spreadsheet = IOFactory::load($file_path);
    $res = [];
    // On va chercher la feuille qui s'appelle "Import.
    $sheetData = $spreadsheet->getSheetByName("Import");

    if (!empty($sheetData)) {
      foreach ($sheetData->getRowIterator(1) as $row) {
        $cellIterator = $row->getCellIterator('A', 'A');
        $cellIterator->setIterateOnlyExistingCells(TRUE);
        foreach ($cellIterator as $cell) {
          if (!empty($cell->getValue())) {
            $res[] = $cell->getValue();
          }
        }
      }
    }
    return $res;
  }
```

- Création du Batch dans le Submit

La batch est exécuté dans la fonction submitForm du formulaire :

Initialisation du Batch

```php
/**
   * {@inheritdoc}
   */
  public function submitForm(array &$form, FormStateInterface $form_state) {
    // On récupère notre fichier uploadé.
    $file = $form_state->getValue(['storage', 'file']);
    $file_path = $this->fileSystem->realpath($file[0]->uri->value);
    // On exécute le traitement du fichier.
    $messages = $this->import($file_path, $context);
    // Initialisation du batch.
    $batch = [
      'title' => t('Importing'),
    // Les opérations seront ajoutés dans le foreach ci-dessous.
      'operations' => [],
      'init_message' => $this->t('Import.'),
      'progress_message' => $this->t('Processing.'),
      'error_message' => $this->t('The process has encountered an error.'),
    // Appellée à la fin du Batch.
      'finished' => [$this, 'finished'],
    ];

    foreach ($messages as $key => $value) {
      // On ajout notre opérations Batch effectué sur chaque ligne importées.
      $batch['operations'][] = [[$this, 'printMessage'], [$key, $value, count($messages), []]];
    }

    batch_set($batch);
  }
```

- Fonctions exécutés par le Batch

Dans cet exemple on exécute 2 fonctions qui sont appellés par le batch

La fonction **printMessage** avec les arguments suivants ⇒ [$key, $value, count($messages), []] que l’on va retrouver dans la signature de la fonction

```php
/**
   * Fonction executée lors du Batch.
   * On affiche juste un Message qui affiche le contenu du fichier
   */
  public function printMessage($line, $value, $count, &$context) {
    if (empty($context['sandbox'])) {
      $context['sandbox']['progress'] = 0;
      $context['sandbox']['max'] = $count;
    }
    else {
      $context['sandbox']['progress'] += 1;
    }
		sleep(2) // pour simuler une action lourde et voir le batch progresser
    \Drupal::messenger()->addMessage('Message ' . ': ' . $line . " => " . $value);

```

Et la fonction **finished** executé à la fin du batch

```php
 /**
   * Finished.
   */
  public function finished() {
    \Drupal::messenger()->addMessage("Le fichier a été traité");
  }
```

- Exécution Test

A l’execution on obtient un batch avec notre barre de progression

Et finalement le résultat final

qui sert à rien mais ça fonctionne

- Conclusion Test

Voilà cet exemple ne sert pas a grand chose mais explique le principe on peut imaginer un cas d’application avec un export csv contenant les id des nodes Drupal et pouvoir exécuter une fonction “lourde” sur ces nodes à l’aide de notre batch.

- Lien github
