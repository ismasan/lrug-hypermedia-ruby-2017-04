## Hypermedia APIs

### And Ruby

^ * Hypermedia / REST broad subject, theoretical
* I'll focus on one aspect (links) and show concrete examples

---

^ disrgarded in the Ruby community?

![fit](assets/dhh.png)

---

^ regular JSON response

```
GET /orders/:id
```

```json
{
  "updated_at": "2017-04-10T18:30Z",
  "id": 123,
  "status": "open",
  "total" 1000
}
```

---

^ actions on resource

```
PUT /orders/:id/place
```

```json
{
  "updated_at": "2017-04-10T18:40Z",
  "id": 123,
  "status": "placed",
  "total" 1000
}
```

---

^ typical Ruby client
hard-coded URLs

```ruby
order = Client.get("/orders/123")

order = Client.put("/orders/123/place")
```

---

^ slightly better
Still need to release new versions every time

```ruby
order = Client.get_order(123)

order = Client.place_order(123)
```

---

^ Need to read docs to undestand workflow

![fit](assets/stripe-docs.png)

---

^ play catch up with API

![fit](assets/github-client.png)

---


![fit](assets/heroku-client.png)

---

![fit](assets/stripe-client.png)

---

# Links

^ central to REST APIs.
defines endpoint, request method, is documented.

```html
<a href="/foobar">Foo bar</a>

<link rel="stylesheet" href="/styles.css" />

<form action="/orders/place" method="PUT">
  ...
</form>
```

---

# Links are actions

^ a link defines an operation or state transition

```html
<form action="/orders/place" method="PUT">
  <input type="text" name="discount_code" value="" />
  <select name="shipping">
    <option value="1">Free shipping</option>
    <option value="2">Expensive shipping</option>
  </select>
</form>
```

---

^ I'll focus on links in APIs
and see what we can do in Ruby

### ~~Hypermedia~~
## APIs with links

---

^ without links

```
GET /orders/:id
```

```json
{
  "udpated_at": "2017-04-10T18:30Z",
  "id": 123,
  "status": "open",
  "total" 1000
}
```

---

^ with links

```
GET /orders/:id
```

```json
{
  "_links": {
    "place_order": {
      "href": "https://api.com/orders/123/place",
      "method": "put"
    }
  },
  "updated_at": "2017-04-10T18:30Z",
  "id": 123,
  "status": "open",
  "total" 1000
}
```

---

```json
  "_links": {
    "place_order": {
      "href": "https://api.com/orders/123/place",
      "method": "put"
    }
  }
```

---

# The client

---

```ruby
class Link
  attr_reader :request_method, :href

  def initialize(attrs, client)
    @request_method = attrs.fetch("method", :get).to_sym
    @href = attrs["href"]
    @client = client
  end
  ...
end
```

---

```ruby
class Link
  ...

  def run(payload = {})
    case request_method
      when :get
        @client.get(href, payload)
      when :put
        @client.put(href, JSON.dump(payload))
      when :delete
        # etc
    end
  end
end
```

---

```ruby
link = Link.new({
  "href": "https://api.com/orders/123/place",
  "method": "put"
}, client)
```

---

```ruby
# PUT https://api.com/orders/123/place
response = link.run
```

---

```ruby
# PUT https://api.com/orders/123/place
response = link.run(
  discount_code: "freestuff"
)
```

---

```ruby
entity = Entity.new(json_data, client)
```

---

```ruby
entity = Entity.new({
  "_links": {
    "place_order": {
      "href": "https://api.com/orders/123/place",
      "method": "put"
    }
  },
  "updated_at": "2017-04-10T18:30Z",
  "id": 123,
  "status": "open",
  "total" 1000
}, client)
```

---

```ruby
class Entity
  def initialize(data, client)
    @data = data
    @links = @data.fetch("_links", {})
    @client = client
  end

  ...
end
```

---

^ don't forget respond_to_missing

```ruby
class Entity
  ...

  def method_missing(method_name, *args, &block)
    if @data[method_name]
      @data[method_name]
    elsif @links[method_name]
      ...
    else
      super
    end
  end
end
```

---


```ruby
def method_missing(method_name, *args, &block)
  ...

  elsif @links[method_name]
    Link.new(
      @links[method_name], @client
    ).run(*args)

  ...
end
```

---

```ruby
order = Entity.new(json_data, client)

order.id # 123
order.status # "open"
```

---

```ruby
# PUT https://api.com/orders/123/place

order.place_order(
  discount_code: "freestuff"
)
```

---


```ruby
class Link
  ...

  def run(payload = {})
    response = case request_method
      when :put
        @client.put(href, JSON.dump(payload))
      when :delete
        # etc
    end

    Entity.new(response.body, client)
  end
end
```

---


```ruby
# PUT https://api.com/orders/123/place

updated_order = order.place_order(
  discount_code: "freestuff"
)

updated_order.status # "placed"
```

---

# Root resource

```
GET /
```

```json
{
  "_links": {
    "orders": {
      "href": "https://...",
    },
    "create_order": {
      "href": "https://...",
      "method": "post"
    }
  }
}
```

---


```ruby
def load(url)
  response = client.get(url)
  Entity.new(response.body, client)
end
```

---

# Workflows

```ruby
api = Client.load("https://api.com")

# create order
order = api.create_order(line_items: [...])

# add items
order.add_line_item(id: "iphone", quantity: 2)

# place it
order = order.place_order
```

---

# Pagination

```ruby
# list orders
orders = api.orders(sort: "total.desc")

orders.each do |order|
  puts order.total
end
```

---

```
GET /orders
```

```json
{
  "_links": {
    "next": {
      "href": "https://api.com/orders?page=2"
     }
   },
   "total_items": 323,
   "items": [
      {"id": 123, "total": 100},
      {"id": 234, "total": 50},
      // ... etc
   ]
}
```

---

```ruby
class Entity
  def initialize(data, client)
    ...
    self.extend EnumerableEntity if data["items"]
  end
end
```

---


```ruby
module EnumerableEntity
  include Enumerable

  def each(&block)
    self.items.each &block
  end
end
```

---


```ruby
page = root.orders

page.each do |order|
  ...
end

page = page.next if page.respond_to?(:next)
```

---

```ruby
def to_enum
  page = self

  Enumerator.new do |yielder|
    loop do
      page.each{|item| yielder.yield item }
      raise StopIteration unless page.respond_to?(:next)
      page = page.next
    end
  end
end
```

---

```ruby
all_orders = root.orders(sort: "total.desc").to_enum

all_orders.each{|o| ...}

all_orders.map(&:total)

all_orders.find_all{|o| o.total > 200 }
```

---

* CLIs
* Workflow scripts
* Chat bots

---

![fit](assets/bootic-docs.png)

---

^ same pattern everywhere

![fit](assets/slack-bot.png)

---

## The server
