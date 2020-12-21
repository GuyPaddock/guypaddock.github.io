---
layout: article
title: "Implementing Custom Processor for Search API 1.x on Drupal 8"
categories: posts
modified: 2020-12-20T20:10:00-04:00
tags: [drupal, search-api, date]
comments: true
ads: false
---
## The Problem
I maintain a proprietary Drupal distribution that was originally built on
Drupal 7 but has recently been rebuilt from scratch on Drupal 8 (and will soon 
be ported to Drupal 9). As part of building the new platform, one of our
stakeholders asked for a feature that we didn't have in the old version --
better handling of searches for date values.

In our platform, we have a field called "Asset date". This field is stored as a
text field -- rather than a traditional Drupal date -- because we support
variable precision. That is to say, the "Asset date" could be `1995`, `1995-05`,
or `1995-05-04`, depending upon what data is available about the item that's
being saved to the database. To make things consistent, we enforce that the
value in this field must be in ISO 8601 format. In other words, it must be in
the format `YYYY-MM-DD`, `YYYY-MM`, or `YYYY` -- no other date formats are
accepted.

The unfortunate result of this scheme was that the only way someone could locate
records in our system by its date was to search for the exact date as it
appeared in the record (e.g. `1995-05-04`). The stakeholder wanted more
flexibility -- he wanted the ability to search for `1995` or `"May 1995"` and
have it return results that started with `1995-05`. In the old system, the way
we used to address requests like this was to provide a date range filter on the
search page, but we felt it might be possible to do better this time around. 

Ultimately, we knew that what we needed was for the search index to contain all
the different permutations of the date value. So, for a date like `1995-05-04`,
we wanted the index to contain `1995-05` and `1995` as alternate matches. Then,
to handle searches like `May 1995` we wanted to have all the variations on date
formats indexed as well. A naive approach to accomplish this would be for our
team to just specify all the different variations in another indexed field --
like descriptions or notes -- but that's a lot of unnecessary labor for a very
specific search use case; not to mention the fact that such a convention is easy 
to forget when adding several records.

## The Solution
How could we get Search API to do this for us? Enter Search API "processors".
Processors allow you to apply transformations to the way that data is indexed
and queried on your site. So, for our use case, we figured we could write one or
more processors that alter each date at indexing time to add all the different
permutations.

## The Processors
In the end, we actually created two processors. One converts ISO 8601 dates into
their various alternative/synonymous representations according to US and
European standards (e.g. dates like `May 1995`, `05/1995`, and `05/04/1995`).
The other converts ISO 8601 dates into their various ISO 8601 permutations 
(`1995-05-04` becomes `1995-05-04`, `1995-05`, and `1995`).

Below is the code for each processor. Since we leverage Apache Solr provided by
Pantheon, we're unfortunately limited to Solr v3.6 which means we're using the
soon-to-be-EOL Search API Solr version `8.x-1.x`. If you are using a newer
version of Search API Solr, your mileage may vary, but hopefully the interface
for processors should still be the same on the Search API side (we wrote these
while using Search API `8.x-1.17`).

### Processor #1 - ISO 8601 Date Synonymizer
This processor converts the date format our team enters into the "Asset date"
field into their more common, colloquial formats used in the United States and
Europe.

The code for this processor can be found below. If you want to use this code,
you'll want to
[create a new custom module](https://www.drupal.org/docs/creating-custom-modules)
and then define the processor in a file called
`src/Plugin/search_api/processor/Iso8601DateSynonymizer.php` inside that module.

```php
<?php

namespace Drupal\YOUR_MODULE\Plugin\search_api\processor;

use Drupal\search_api\Processor\FieldsProcessorPluginBase;

/**
 * Adds synonyms/alternate date formats for ISO 8601 dates.
 *
 * @SearchApiProcessor(
 *   id = "iso8601_date_synonymizer",
 *   label = @Translation("ISO 8601 date synonymizer"),
 *   description = @Translation("Adds alternative representations of ISO 8601-formatted dates at index time. For example, <em>1986-04-01</em> gets indexed as both <em>1986-04-01</em> and <em>April 1, 1986</em>. If using the 'ISO 8601 Date tokenizer', make sure this is invoked before it."),
 *   stages = {
 *     "pre_index_save" = -1,
 *     "preprocess_index" = -1,
 *   }
 * )
 */
class Iso8601DateSynonymizer extends FieldsProcessorPluginBase {

  /**
   * Matches ISO 8601 dates in the format YYYY-MM-DD, YYYY-MM, or YYYY.
   *
   * Each date component is captured in a named group.
   */
  const REGEX_ISO_8601_DATE_WITH_CAPTURE =
    '/^(?<year>[0-9]{4})(?>-(?<month>1[0-2]|0[1-9])(?>-(?<day>3[01]|0[1-9]|[12][0-9]))?)?$/';

  /**
   * {@inheritdoc}
   */
  protected function testType($type) {
    // Only apply to "text" types since we return the synonyms as distinct,
    // space-separated "words" (e.g. "1986-Apr-01 Apr-1986"), so in order for
    // searches to match, the index needs to treat this as full-text. Otherwise,
    // the user would have to search for *exactly* "1986-Apr-01 Apr-1986",
    // which isn't helpful.
    return $this->getDataTypeHelper()->isTextType($type);
  }

  /**
   * {@inheritdoc}
   */
  protected function process(&$value) {
    // In the absence of the tokenizer processor, this ensures split words.
    $words =
      preg_split('/\s/', strip_tags($value), -1, PREG_SPLIT_NO_EMPTY);

    $new_words = [];

    foreach ($words as $word) {
      $date_synonyms = $this->getDateSynonyms($word);

      // Keep original, unmodified value.
      $new_words[] = $word;

      if (!empty($date_synonyms)) {
        $new_words[] = implode(' ', $date_synonyms);
      }
    }

    $value = implode(' ', $new_words);
  }

  /**
   * Gets all the alternative ways to format the given date string.
   *
   * @param string $value
   *   A date string for which synonyms/alternate formats are desired.
   *
   * @return string[]
   *   An array containing all the alternate ways to format the given date. For
   *   example:
   *   @code
   *   [
   *     '04/01/1986',
   *     '04/01/86',
   *     '01/04/1986',
   *     '01/04/86',
   *     '01-Apr-1986',
   *     '01 Apr 1986',
   *     'April 01, 1986',
   *     'Apr-1986',
   *     'Apr 1986',
   *     'April 1986',
   *   ]
   * @endcode
   */
  protected function getDateSynonyms(string $value): array {
    $synonyms = [];

    if (preg_match(self::REGEX_ISO_8601_DATE_WITH_CAPTURE, $value, $matches)) {
      $month_number = $matches['month'] ?? NULL;
      $day_number   = $matches['day'] ?? NULL;

      if ($day_number !== NULL) {
        $date = \DateTime::createFromFormat('!Y-m-d', $value);

        // US - "04/01/1986".
        $synonyms[] = $date->format('m/d/Y');

        // US - "04/01/86".
        $synonyms[] = $date->format('m/d/y');

        // EU - "01/04/1986".
        $synonyms[] = $date->format('d/m/Y');

        // EU - "01/04/86".
        $synonyms[] = $date->format('d/m/y');

        // "01-Apr-1986".
        $synonyms[] = $date->format('d-M-Y');

        // "01 Apr 1986".
        $synonyms[] = $date->format('d M Y');

        // "April 01, 1986".
        $synonyms[] = $date->format('F d, Y');
      }
      elseif ($month_number !== NULL) {
        $date = \DateTime::createFromFormat('!Y-m', $value);
      }

      if (isset($date)) {
        // "Apr-1986".
        $synonyms[] = $date->format('M-Y');

        // "Apr 1986".
        $synonyms[] = $date->format('M Y');

        // "April 1986".
        $synonyms[] = $date->format('F Y');
      }
    }

    return $synonyms;
  }

}
```

### Processor #2 - ISO 8601 Date Tokenizer
This processor converts each date into the various alternate search tokens that
should match on the date value.

The code for this processor can be found below. This should go in a file called
`src/Plugin/search_api/processor/Iso8601DateTokenizer.php` inside your custom
module.

```php
<?php

namespace Drupal\YOUR_MODULE\Plugin\search_api\processor;

use Drupal\search_api\Processor\FieldsProcessorPluginBase;

/**
 * Tokenizes ISO 8601 dates into YYYY-MM-DD, YYYY-MM, and YYYY formats.
 *
 * @SearchApiProcessor(
 *   id = "iso8601_date_tokenizer",
 *   label = @Translation("ISO 8601 date tokenizer"),
 *   description = @Translation("Tokenizes ISO 8601-formatted dates at index time. For example, <em>1986-04-01</em> gets indexed as <em>1986-04-01</em>, <em>1986-04</em>, and <em>1986</em>."),
 *   stages = {
 *     "pre_index_save" = 0,
 *     "preprocess_index" = 0,
 *   }
 * )
 */
class Iso8601DateTokenizer extends FieldsProcessorPluginBase {

  /**
   * Matches ISO 8601 dates in the format YYYY-MM-DD, YYYY-MM, or YYYY.
   */
  const REGEX_ISO_8601_DATE =
    '/^([0-9]{4})(?>-(1[0-2]|0[1-9])(?>-(3[01]|0[1-9]|[12][0-9]))?)?$/';

  /**
   * {@inheritdoc}
   */
  protected function testType($type) {
    // Only apply to "text" types since we return the permutations as distinct,
    // space-separated "words" (e.g. "1986-04-01 1986-04 1986"), so in order for
    // searches to match, the index needs to treat this as full-text. Otherwise,
    // the user would have to search for *exactly* "1986-04-01 1986-04 1986",
    // which isn't helpful.
    return $this->getDataTypeHelper()->isTextType($type);
  }

  /**
   * {@inheritdoc}
   */
  protected function process(&$value) {
    // In the absence of the tokenizer processor, this ensures split words.
    $words =
      preg_split('/\s/', strip_tags($value), -1, PREG_SPLIT_NO_EMPTY);

    $new_words = [];

    foreach ($words as $word) {
      $date_permutations = $this->getDatePermutations($word);

      if (empty($date_permutations)) {
        // Pass through unmodified.
        $new_words[] = $word;
      }
      else {
        $new_words[] = implode(' ', $date_permutations);
      }
    }

    $value = implode(' ', $new_words);
  }

  /**
   * Gets all the substrings that should match on the given date string.
   *
   * @param string $value
   *   A date string for which token permutations are desired.
   *
   * @return string[]
   *   All token permutations for the given date. For example:
   *   @code
   *   [
   *    '1986-04-01',
   *    '1986-04',
   *    '1986',
   *   ]
   *   @endcode
   */
  protected function getDatePermutations(string $value): array {
    $permutations = [];

    if (preg_match(self::REGEX_ISO_8601_DATE, $value, $matches)) {
      $date_parts       = array_slice($matches, 1);
      $total_part_count = count($date_parts);

      for ($perm_index = 0; $perm_index < $total_part_count; ++$perm_index) {
        $permutations_parts =
          array_slice($date_parts, 0, $total_part_count - $perm_index);

        $permutations[] = implode('-', $permutations_parts);
      }
    }

    return $permutations;
  }

}
```

## Enabling the Processors
After creating a custom module, adding code to the appropriate files, and
adjusting the namespace names in the files to match the name of our module,
we are ready to test out the new processors.

### Step 1: Enable Your Custom Module or Clear Caches
Before you can set them up, you need to
[enable your custom module](https://www.drupal.org/docs/extending-drupal/installing-modules#s-step-2-enable-the-module).
If you added the processors to a module that was already installed (or you
later modify any of the information in the `SearchApiProcessor` annotation),
you'll want to
[clear caches/rebuild Drupal's registry](https://www.drupal.org/docs/user_guide/en/prevent-cache-clear.html).

### Step 3: Configure Date Text Fields for Indexing
Next, you will want to ensure that your date field(s) are indexed as "Fulltext
Unstemmed" on the "Fields" tab of your search index
(`/admin/config/search/search-api/index/YOUR_INDEX/fields`). If you index them
as "String" values, searches won't work properly.

**Also note:** these processors were written to work only with _text_ fields.
In other words, we wrote them to handle single-value "Text" fields in our 
Drupal entities, and the processors expect that the text will be formatted in
ISO 8601 date format. We did not write them or test them to work with standard
Drupal date fields nor for dates that are stored in a different format in your
text fields. If you need this, you will need to customize the processor code.

### Step 2: Enable and Configure the New Processors
Now, visit the "Processors" tab of your search index
(e.g. `/admin/config/search/search-api/index/YOUR_INDEX/processors`)
and enable both the "ISO 8601 date synonymizer" and "ISO 8601 date tokenizer"
processors. Make sure that the synonymizer runs BEFORE the tokenizer (i.e. in
the "Preprocess index" under "Processor order", the "ISO 8601 date synonymizer"
should appear above the "ISO 8601 date tokenizer" in the list). Then, under
"Processor settings", visit the vertical tab for each processor and uncheck all
fields except your date field(s).

### Step 4: Re-index content
After making all these changes, remember to re-index your content.

### Step 5: Test!
It's finally time to try out a few searches. For a record containing a date like
`1986-04-01`, all of the following searches should match it:

- `1986-04-01`
- `1986-04`
- `1986`
- `04/01/1986`
- `04/01/86`
- `01/04/1986`
- `01/04/86`
- `01-Apr-1986`
- `01 Apr 1986`
- `April 01, 1986`
- `Apr-1986`
- `Apr 1986`
- `April 1986`

You may notice that formats like "1/4/1986" and "4/1/1986" aren't supported.
This is intentional -- the synonymizer doesn't expose one-digit months or days
because Solr only indexes words or numbers that are at least two digits.
