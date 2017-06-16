# Where in the World is ~~Carmen Sandiego~~ Gowie?

Spending some my 20's traveling was always a dream of mine since I first went abroad to visit my sister in Barcelona when I was 14, so towards the end of 2016 I quit my job managing a software team, started my own software consulting biz, and booked a one-way flight abroad. While preparing for that trip and searching for my first client, I developed a couple little apps that allowed me to track myself as I hopped from country to country. This automated the mundane process of me having to manually type into my website, "This week I'm in Rishikesh, India" and it also gave me a database that shows I went from Thailand, to Malaysia, to Turkey, and so on. You can see the result of these apps in action on [my website](https://mattgowie.com) where I share my current location.

I liked building this project and I thought it would be cool to break down how I did it for folks who might be interested. Read on if you're one of those folks!

## Grabbing my Location

To properly track my current location in the world, I needed a device that came with me everywhere. My MacBook-Pro (named GowieBook-Pro) and iPhone both fit the bill for this so I decided to write a small OSX app that did the job. I ended up going with a Mac Menubar application because it made sense to build something with not much of a UI. I aptly termed that project [WhereAmIMenuBar](https://github.com/Gowiem/WhereAmIMenuBar) and it's dead simple. It's a small Swift app that grabs the current IP address of my machine, fetches the location of that IP from http://ip-api.com, and then POSTs it as a small JSON blob to a small web service that we'll get to later. Here is the [small bit of code](https://github.com/Gowiem/WhereAmIMenuBar/blob/master/WhereAmIBar/WhereAmI.swift#L25) that'll give you a gist of what I'm talking about:

```swift
func updateLocation(previousIp: String?) -> Promise<[String: Any]> {
    return Promise { fullfil, reject in
        firstly {
            return fetchIp()
        }.then { ip -> Promise<[String: Any]> in
            NSLog("Previous IP: \(previousIp) IP: \(ip)")
            if previousIp == nil || previousIp! != ip {
                return self.fetchLocation(ipAddress: ip)
            } else {
                throw WhereAmIError.NoReasonToUpdateCancel
            }
        }.then { locationJson in
            return self.postLocationUpdate(locationInfo: locationJson)
        }.then { locationInfo in
            fullfil(locationInfo)
        }.catch { error in
            reject(error)
        }
    }
}
```

This [runs every hour](https://github.com/Gowiem/WhereAmIMenuBar/blob/master/WhereAmIBar/AppManager.swift#L51), but really only POSTs my location JSON if my IP address changes. It also includes a small MenuBar UI (who would have guessed?) to allow me to update manually. I set this up so that it is one of the startup items on my MacBook and now my Mac broadcasts my location to my web service whenever I open up my MacBook at a new location. Pretty cool, huh?



## Storing my Location

Okay, so I've got an app that broadcasts my location, but where does that end up? Well it ends up on the web of course. Another small app that I've dubbed [whereami.com](https://github.com/Gowiem/whereami.com) serves as an API for my location data. At the time of writing this I was interested in learning more about the Elixir programming language so I wrote this app in Elixir and used [Plug](https://github.com/elixir-lang/plug) as my web framework. Plug is the equivalent of Ruby's Sinatra/Rack in the Elixir world. Here is the 5 lines of code which accomplish accepting that location data:

```elixir
@doc """
/location
An endpoint to allow the WhereAmIMenuBar application to upload its location information.
This information is stored in a Redis List.
"""
post "/location" do
  body_json =  Poison.Encoder.encode(conn.body_params, [])
  :poolboy.transaction(:redis_pool, fn(worker) -> Redix.command(worker, ["RPUSH", "locations", body_json]) end)
  send_resp(conn, 200, "Success!")
end
```

So here we accepts the incoming location JSON data from the MenuBar application, throw it into a Redis list to be retrieved later, and then serve up a 200. Simple, amirite?

## Fetching my Location

My location is now stored in Redis, but how do we get it out? 
