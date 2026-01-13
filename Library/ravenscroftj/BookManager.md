---
name: Library/myuser/My Library
tags: meta/library
---
  
  This implements my super awesome hello world library!

```space-lua
config.set("BookPrefix", "Media DB/books/")
```
  
```space-lua

-- ==============================
--  Open Library → SilverBullet
-- ==============================

OpenLibrary = {}

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

function OpenLibrary.formatBookPage(book)
  local title = book.title or "Untitled"
  local authors = book.author_name and table.concat(book.author_name, ", ") or "Unknown"
  local year = book.first_publish_year or "n/a"
  local isbn = book.isbn and book.isbn[1] or "n/a"
  local olid = book.key or ""
  local url = "https://openlibrary.org" .. olid
  
  return string.format([[
# %s

**Author(s):** %s  
**Published:** %s  
**ISBN:** %s  
**Open Library:** [View Book](%s)

## Notes


## Quotes

]], title, authors, year, isbn, url)
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
    
    local title = string.gsub(book.title, "[^%w%s%-]", "")
    title = string.gsub(title, "%s+", " ")

    local pageName =  config.get("BookPrefix","Books/") .. title

    print("PageName= "..pageName)
    
    local content = OpenLibrary.formatBookPage(book)
    
    -- Create the page
    space.writePage(pageName, content)
    editor.navigate(pageName)
    editor.flashNotification("Book added: " .. title)
  end
}
```
