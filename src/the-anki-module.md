# The 'anki' Module

All access to your collection and associated media go through a Python
package called `anki`, located in
[pylib/anki](https://github.com/ankitects/anki/tree/main/pylib/anki)
in Anki's source repo.

## The Collection

All operations on a collection file are accessed via a `Collection`
object. 

If running code inside an add-on, the Anki collection will be 
accessible via the Anki main window, called `mw`, as shown below.

**Access collection from mw:**
```python
from aqt import mw


col = mw.col
```
When using the `anki` module outside of Anki, the `mw` object will not exist.
You will need to create your own `Collection` object 
from the `collection.anki2` file. The file can located via the [the docs](https://docs.ankiweb.net/files.html)
or as shown below.

**Initiate collection from file:**
```python
from anki.collection import Collection
from aqt.profiles import ProfileManager

col_path =  ProfileManager.get_created_base_folder(None)
col = Collection(col_path)
```

Some basic examples of what you can do with a collection follow. 
With `mw` usage, please note that you should put its calls 
in something like [testFunction()](./a-basic-addon.md); you can’t run them
directly in an add-on, as add-ons are initialized during Anki startup, before
any collection or profile has been loaded.

Also please note that accessing the collection directly can lead to the UI
temporarily freezing if the operation doesn't complete quickly - in practice
you would typically run the code below in a background thread.

**Get a due card:**

```python
card = mw.col.sched.getCard()
if not card:
    # current deck is finished
```

**Answer the card:**

```python
mw.col.sched.answerCard(card, ease)
```

**Edit a note (append " new" to the end of each field):**

```python
note = card.note()
for (name, value) in note.items():
    note[name] = value + " new"
mw.col.update_note(note)
```

**Get card IDs for notes with tag x:**

```python
ids = mw.col.find_cards("tag:x")
```

**Get question and answer for each of those ids:**

```python
for id in ids:
    card = mw.col.get_card(id)
    question = card.question()
    answer = card.answer()
```

**Make reviews due tomorrow**

```python
ids = mw.col.find_cards("is:due")
mw.col.sched.set_due_date(ids, "1")
```

**Import a text file into the collection**

Requires Anki 2.1.55+.

```python
from anki.collection import ImportCsvRequest
from aqt import mw


path = "/home/dae/foo.csv"
metadata = col.get_csv_metadata(path=path, delimiter=None)
request = ImportCsvRequest(path=path, metadata=metadata)
response = col.import_csv(request)
print(response.log.found_notes, list(response.log.updated), list(response.log.new))
```

Almost every GUI operation has an associated function in anki, so any of
the operations that Anki makes available can also be called in an
add-on.

## Reading/Writing Objects

Most objects in Anki can be read and written via methods in pylib.

```python
card = col.get_card(card_id)
card.ivl += 1
col.update_card(card)
```

```python
note = col.get_note(note_id)
note["Front"] += " hello"
col.update_note(note)
```

```python
deck = col.decks.get(deck_id)
deck["name"] += " hello"
col.decks.save(deck)

deck = col.decks.by_name("Default hello")
...
```

```python
config = col.decks.get_config(config_id)
config["new"]["perDay"] = 20
col.decks.save(config)
```

```python
notetype = col.models.get(notetype_id)
notetype["css"] += "\nbody { background: grey; }\n"
col.models.save(note)

notetype = col.models.by_name("Basic")
...
```

You should prefer these methods over directly accessing the database,
as they take care of marking items as requiring a sync, and they prevent
some forms of invalid data from being written to the database.

For locating specific cards and notes, col.find_cards() and
col.find_notes() are useful.

## The Database

:warning: You can easily cause problems by writing directly to the database.
Where possible, please use methods such as the ones mentioned above instead.

Anki’s DB object supports the following functions:

**scalar() returns a single item:**

```python
showInfo("card count: %d" % mw.col.db.scalar("select count() from cards"))
```

**list() returns a list of the first column in each row, e.g.\[1, 2,
3\]:**

```python
ids = mw.col.db.list("select id from cards limit 3")
```

**all() returns a list of rows, where each row is a list:**

```python
ids_and_ivl = mw.col.db.all("select id, ivl from cards")
```

**execute() can also be used to iterate over a result set without
building an intermediate list. eg:**

```python
for id, ivl in mw.col.db.execute("select id, ivl from cards limit 3"):
    showInfo("card id %d has ivl %d" % (id, ivl))
```

**execute() allows you to perform an insert or update operation. Use
named arguments with ?. eg:**

```python
mw.col.db.execute("update cards set ivl = ? where id = ?", newIvl, cardId)
```

Note that these changes won't sync, as they would if you used the functions
mentioned in the previous section.

**executemany() allows you to perform bulk update or insert operations.
For large updates, this is much faster than calling execute() for each
data point. eg:**

```python
data = [[newIvl1, cardId1], [newIvl2, cardId2]]
mw.col.db.executemany(same_sql_as_above, data)
```

As above, these changes won't sync.

Add-ons should never modify the schema of existing tables, as that may
break future versions of Anki.

If you need to store addon-specific data, consider using Anki’s
[Configuration](addon-config.md#config-json) support.

If you need the data to sync across devices, small options can be stored
within mw.col.conf. Please don’t store large amounts of data there, as
it’s currently sent on every sync.
