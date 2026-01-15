---
name: Library/ravenscroftj/BookManager
tags: meta/library
description: support adding notes about books using metadata from OpenLibrary
---

## Configuration

By default the book information will be stored in `Books/` but you can override this by setting `bookmanager.prefix` in your config:

```lua
config.set("bookmanager.prefix", "Media DB/books/")
```

The filename will be `Book Title (year of publication).md` e.g. `Mistborn (2006).md` by default. You can override this by setting the `bookmanager.filenameTemplate`. 

```lua
config.set("bookmanager.filenameTemplate", [==[${title} (${first_publish_year})]==] )
```

For more information see [Templates in Silverbullet](https://silverbullet.md/API/template) and for reference RE: parameters see [OpenLibrary search API docs](https://openlibrary.org/dev/docs/api/search) and [schema](https://github.com/internetarchive/openlibrary/blob/b4afa14b0981ae1785c26c71908af99b879fa975/openlibrary/plugins/worksearch/schemes/works.py#L38-L91).

```lua
config.set("bookmanager.bookTemplate", 
[==[---
type: book
title: ${title}
year: ${first_publish_year}
dataSource: OpenLibraryAPI
url: https://openlibrary.org/${olid}
id: ${olid}
author: ${author_name}
tags: mediaDB/book
isbn {$isbn[1]}
---]==] )
```


## Implementation


```space-lua

-- ==============================
--  Open Library → SilverBullet
-- ==============================

OpenLibrary = {}
OpenLibrary.defaultFilenameTemplate = [==[${title} (${first_publish_year})]==]

OpenLibrary.defaultPageTemplate = [==[---
type: book
title: ${title}
year: ${first_publish_year}
dataSource: OpenLibraryAPI
url: https://openlibrary.org/${olid}
id: ${olid}
author: 
  - ${author_name}
tags: mediaDB/book
isbn: ${isbn}
---
]==]

function OpenLibrary.searchBook(queryString)
  -- Search by title, author, or ISBN
  local url = "https://openlibrary.org/search.json?q=" .. 
              string.gsub(queryString, " ", "+")
  
  local resp = net.proxyFetch(url, {
    method = "GET",
    headers = { Accept = "application/json" }
  })

  if not resp.ok then
    error("Failed to fetch from Open Library: " .. resp.status)
  end
  
  local data = resp.body

  if #data.docs == 0 then
    editor.flashNotification("No book found for: " .. queryString)
    return nil
  end

  options = {}
  -- show prompt to select docs
  for i, doc in ipairs(data.docs) do
    options[i] = {
      name=doc.title, 
      value=i, 
      description=string.format("By %s, published %s", doc.author_name, doc.first_publish_year)
  }  
  end
  
  local result = editor.filterBox("Select:", options)

  if not result then
    editor.flashNotification("Cancelled book search")
    return nil
  end
  
  return data.docs[result.value]  -- return first match
end

command.define {
  name = "Open Library: Add Book to Knowledge Base",
  run = function()
    local queryString = editor.prompt("Search for book (title, author, or ISBN):")
    if not queryString then
      return
    end

    print("Searching for book " .. queryString)
    
    local book = OpenLibrary.searchBook(queryString)
    if not book then
      return
    end
    
    -- local title = string.gsub(book.title, "[^%w%s%-]", "")
    -- title = string.gsub(title, "%s+", " ")
    local titleTemplate = template.new(config.get("bookmanager.filenameTemplate", OpenLibrary.defaultFilenameTemplate))

    local safeBook = {}

    for k in {'title','first_publish_year','author_name', 'isbn'} do
      safeBook[k] = book[k] or "Unknown"
    end

    safeBook['olid'] = book['key']

    local title = titleTemplate(safeBook)
    local pageName =  config.get("bookmanager.prefix","Books/") .. title

    if not space.pageExists(pageName) then
      local pageTemplate = template.new(config.get('bookmanager.bookTemplate', OpenLibrary.defaultPageTemplate))
      
      local content = pageTemplate(safeBook)
      
      -- Create the page
      space.writePage(pageName, content)
      editor.flashNotification("Book added: " .. title)
    else
      editor.flashNotification("Book already in collection: " .. title)
    end
    

    editor.navigate(pageName)

  end
}
```


