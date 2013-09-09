---
layout: post
title: "Rails API documentation"
date: 2013-07-24 12:46
comments: true
categories: [rails, api, documentation]
---

Lately I've been working on a new project for mobile called Playround. My job is to design and implement the API. I must say that with the help of Rails 4 and Rails API the experience has been the smoothest possible. With requests and models specs I can test the whole application at blazing speeds.
At a certain point however I faced an issue: documentation.

Especially in the first stages of development, working inside a team mandates transparency about the current status of the API so the mobile developers know exactly what to expect from the server while testing locally.
Of course documenting is exceptionally useful also for the mature stage of the project when we will have to publish the API documentation in a beautiful layout. In order to achieve documentation nirvana I started experimenting various ways of building it, possibly in a way that would output something I can use for our public doc.

At first I started putting "debugger" in every test and printing the output of the response, however this task got tedious pretty fast. Looking around I found a [gem][0] that compiles a documentation, however it forced me to use a specific dsl which means I had to rewrite my tests. When I started I used the convention adopted by the first requests tests you find with scaffolds and I wanted to keep that.

In order to achieve this I wrote a simple script in the spec helper that does the following things:

- For every request spec file it creates a corresponding txt file inside the doc folder.
- For each test the path, status, request and response are written inside the corresponding file.

Request tests have to follow a convention:

- Top level descriptions are named after the model (plural form) followed by the word "Requests". For the model Arena it would be "Arenas Requests".
- Actions are in the form of "VERB path". For the show action of the Arenas controller it would be "GET /arenas/:id".

The code:

```ruby
config.after(:each, type: :request) do
  if response
    example_group = example.metadata[:example_group]
    example_groups = []

    while example_group
      example_groups << example_group
      example_group = example_group[:example_group]
    end

    action = example_groups[-2][:description_args].first if example_groups[-2]
    example_groups[-1][:description_args].first.match(/(\w+)\sRequests/)
    file_name = $1.underscore

    File.open(File.join(Rails.root, "/docs/#{file_name}.txt"), 'a') do |f|
      f.write "#{action} \n\n"

      request_body = request.body.read

      if request.headers['Authorization']
        f.write "Headers: \n\n"
        f.write "Authorization: #{request.headers['Authorization']} \n\n"
      end

      if request_body.present?
        f.write "Request body: \n\n"
        f.write "#{JSON.pretty_generate(JSON.parse(request_body))} \n\n"
      end

      f.write "Status: #{response.status} \n\n"

      if response.body.present?
        f.write "Response body: \n\n"
        f.write "#{JSON.pretty_generate(JSON.parse(response.body))} \n\n"
      end
    end unless response.status == 401 || response.status == 403 || response.status == 301
  end
end
```

Example output for "Rounds Requests" POST action:

```
POST /v1/rounds

Headers:

Authorization: Token token="36260243e091bfe56f96483592afc723"

Request body:

{
  "round": {
    "game_name": "dota2",
    "arena": {
      "foursquare_id": "5104"
    }
  }
}

Status: 201

Response body:

{
  "round": {
    "id": "ec6add8b-709f-475d-8f06-8ad44d8a95d3",
    "state": "waiting_for_players",
    "created_at": "2013-07-24T12:16:14.700Z",
    "game": {
      "id": "1c59b30e-599a-4ea1-9d5c-a364079528ad",
      "name": "dota2",
      "display_name": "Dota 2",
      "picture_url": "http://localhost:8080/assets/dota2.jpg",
      "teams": [
        {
          "name": "radiant",
          "display_name": "Radiant",
          "number_of_players": 5
        },
        {
          "name": "dire",
          "display_name": "Dire",
          "number_of_players": 5
        }
      ]
    },
    "arena": {
      "id": "5c593125-a114-4a1a-936f-2cc4b21fa0a8",
      "name": "Clinton St. Baking Co. & Restaurant",
      "latitude": 40.721294,
      "longitude": -73.983994,
      "foursquare_id": "5104"
    },
    "teams": [

    ],
    "user": {
      "id": "87fb0fe4-2f0c-400d-ba00-000c3f5ea642",
      "name": "Test User",
      "picture_url": "http://graph.facebook.com/12132/picture?type=square",
      "facebook_id": "12132"
    }
  }
}
```

I'm excluding 401, 403 and 301 status codes because those cases are grouped and documented inside a common area in my documentation, but there is nothing special about them.

Now to the beautiful layout part. Right now I'm copy pasting those response and requests bodies inside the templates of a [Jekyll application][1] hosted on [Github Pages][2]. One way to automate this would be to use a templating language in order to output html documents instead of plain txt files. Since the production documentation should change way less frequently than the development one, this is a automation I can skip for now. It's far more important to keep a fresh copy of the features for internal usage, which can be rebuilt anytime by anyone with no effort.

[0]: https://github.com/zipmark/rspec_api_documentation
[1]: https://github.com/playround/playround-developer
[2]: http://developer.goplayround.com