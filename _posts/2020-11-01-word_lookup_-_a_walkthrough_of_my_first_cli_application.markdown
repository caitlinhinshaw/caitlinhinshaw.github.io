---
layout: post
title:      "Word Lookup - A Walkthrough of My First CLI Application"
date:       2020-11-01 06:03:00 +0000
permalink:  word_lookup_-_a_walkthrough_of_my_first_cli_application
---


Word Lookup is a CLI application coded in Ruby that I created for my first project at Flatiron School.

It uses the [Words API](https://rapidapi.com/dpventures/api/wordsapi) to look up a word provided by the user and list its definitions. The user can then choose to view the synonyms, antonyms, rhymes, or similar words for the word they entered.

This post goes through the app from start to finish and explains the classes and methods used, as well as some of the design decisions I made. It's mostly for my own future reference, but hopefully it will be of some use to other newbies looking to build their first CLI app!

## Source code

Check out the source code for this project at <https://github.com/caitlinhinshaw/word-lookup>.

## Starting the program

To start the app, navigate to the root folder in the terminal, then run `./bin/word_lookup`. This file contains the following code:

```ruby
#!/usr/bin/env ruby

require_relative '../lib/environment'

WordLookup::CLI.new.call
```

The top line tells the terminal that this is a Ruby file - by including it, we don't have to put 'ruby' in front of the filename to run it on the command line (i.e. `ruby ./bin/word_lookup`).

The middle line loads `environment.rb`, which in turn loads all the other needed files and gems:

```ruby
require_relative "./word_lookup/version"
require_relative "./word_lookup/cli"
require_relative "./word_lookup/word"
require_relative "./word_lookup/api"

require 'pry'
require 'json'
require 'uri'
require 'net/http'
require 'openssl'
require 'dotenv/load'
require 'colorize'
```

The last line starts the CLI sequence by creating a new instance of the CLI class and calling the CLI instance method `call`.

```ruby
#CLI Class
def call
    puts "\nWelcome to Word Lookup!".colorize(:light_cyan)
    choose_word
end
```

We can see that the `call` method prints out a greeting to the user, then calls the `choose_word` method.

```ruby
#CLI class
def choose_word
    puts "\nEnter a word to look up:".colorize(:light_green)
    word = gets.strip
    @current_word = WordLookup::Word.new(word)
    validate_word
end
```

The `choose_word` method asks the user what word they want to look up and saves their response into a local variable `word`.

Then, it creates a new instance of the Word class by passing in `word` as an argument, and saves this instance to `@current_word`, an instance variable of the CLI class.

Finally, it calls the next method in the CLI flow, `validate_word`.

## Creating a new instance of the Word class

Since we want to fetch information from the Words API when a Word instance is created, we need to create a custom `initialize` method.

```ruby
#Word class
def initialize(word_text)
    @word_text = word_text
    @detail_hash = WordLookup::API.new.fetch_word_details_from_API(@word_text)
    @@all << self
end
```

The custom method accepts the `word_text` that the user entered and saves it to the instance attribute `@word_text`.

It creates a new instance of the API class, calls the method `fetch_word_details_from_API` on the new API instance with `@word_text` as the argument, then saves the returned hash into the attribute `@detail_hash`.

Finally, it saves the new Word instance (accessed with `self`) to
the class attribute `@@all`.

### Fetching word details from the API

The API class uses the `uri`, `openssl`, and `dotenv` gems, as well as the `net/http` and `json` packages included with Ruby.

When `fetch_word_details` is called, it calls the method `API_call`, which returns JSON data. The method `convert_json_to_ruby` converts the JSON into Ruby, then `fetch_word_details` returns the converted hash back to the Word method that called it.

```ruby
#API class
def fetch_word_details_from_API(word)
    url = URI("https://wordsapiv1.p.rapidapi.com/words/#{word}/")

    json = API_call(url)

    convert_json_to_ruby(json)
end

def API_call(url)
    http = Net::HTTP.new(url.host, url.port)
    http.use_ssl = true
    http.verify_mode = OpenSSL::SSL::VERIFY_NONE

    request = Net::HTTP::Get.new(url)
    request["x-rapidapi-host"] = 'wordsapiv1.p.rapidapi.com'
    request["x-rapidapi-key"] = ENV['WORDS_API_KEY']

    http.request(request)
end

def convert_json_to_ruby(json)
    JSON.parse(json.body)
end
```

## Validating the word

Before trying to return any details on the word the user entered, we should make sure it's a real word that exists in the API data, not something like 'asfadsfa' or '431&!6'.

The Words API makes this easy: it includes a `"success"` key in the
returned hash. If the value is `false`, then the word was not found in the database. We can use this to build our `validate_word` method.

```ruby
#CLI class
def validate_word
    if @current_word.detail_hash["success"] == false
        puts "\nThe word '#{@current_word.word_text}' was not found. Please try again.".colorize(:light_red)
        choose_word
    else
        @current_word.add_details
        list_definitions
    end
end
```

If the word is not valid, it calls the `choose_word` method again, and the user can enter another word.

If the word is valid, it calls the `add_details` method from the Word class on the `@current_word` instance, which assigns values to `@current_word`'s attributes based on the data in the `@detail_hash`.

Then, it calls the next method in the CLI flow - `list_definitions`.

### Populating the Word instance with attributes

```ruby
#Word class
def add_details
    fetch_definitions
    fetch_synonyms
    fetch_antonyms
    fetch_similar_words
    fetch_rhymes
end

def fetch_definitions
    @definitions = []
    @detail_hash["results"].each do |meaning|
        @definitions << meaning["definition"]
    end
end

def fetch_synonyms
    @synonyms = []
    @detail_hash["results"].each do |meaning| #collects the synonyms for each meaning to display all at once
        @synonyms << meaning["synonyms"]
        @synonyms << meaning["also"]
    end
    @synonyms = flatten_and_remove_nils(@synonyms)
end

#Other fetch methods have a similar structure to the above
```

Each attribute for an instance of Word is an array that lists all the details found for a particular detail type: i.e., all the synonyms, all the rhymes, etc.

Every fetch method iterates through the `@detail_hash` and adds
values from the relevant key-value pairs to the attribute array.

Sometimes, the values added are an array themselves, which turns the attribute array into a nested array. Since we just want a simple list of all the details, we can use a method to flatten the attribute array and make sure it doesn't contain any nulls or duplicates.

```ruby
#Word class
def flatten_and_remove_nils(array)
    flat_array = array.flatten.uniq
    flat_array.compact unless array.none? {|a| a.nil?}
    #returns a flattened array with no nils or duplicate values
    #unless statement is necessary because array.compact returns nil if
    #there are no nil values, and we want it to return the array instead
end
```

After all the details are added to the Word instance, we can easily access them from inside the API class.

## Listing the definitions

```ruby
#CLI class
def list_definitions
    puts "\nHere are the definition(s) for '#{current_word.word_text}':".colorize(:light_blue)
    @current_word.definitions.each_with_index do |definition, index|
        puts "#{index+1}. #{definition}"
    end
    list_detail_categories
end
```

The method `list_definitions` gets the current word's `@definitions` array and iterates through the array to print each definition to a numbered list.

After the definitions are printed, it calls `list_detail_categories`, the next method in the CLI flow.

## Listing the detail categories

```ruby
#CLI class
def list_detail_categories
    puts "\nWhat information would you like about the word '#{current_word.word_text}'? Enter a number.".colorize(:light_green)
    puts "\n"
    WordLookup::Word.detail_categories.each_with_index do |category, index|
        puts "#{index+1}. #{category}"
    end
    choose_details
end
```

The `list_detail_categories` method prompts the user to choose what kind of information they would like about the chosen word.

It prints the available options using the Word class variable
`@@detail_categories`, which contains the available detail types as strings in an array. Storing the names of available detail types this way makes it easy to print them as a numbered list using `each_with_index`, and also increases maintainability since those string values are not repeated throughout the code.

Finally, this method calls the next method in the CLI flow, `choose_details`.

## Choosing the details

```ruby
#CLI class
def choose_details
    chosen_details = gets.strip.to_i
    if valid_choice?(chosen_details, WordLookup::Word.detail_categories)
        list_details(chosen_details)
    else
        puts "This is not a valid choice. Please try again.".colorize(:light_red)
        choose_details
    end
end
```

This method takes in the user's input and saves it to the local variable `chosen_details`.

Since there are only 4 valid choices - 1, 2, 3, and 4 - we need to use another method, `valid_choice?` to validate the user's input.

If the choice is valid, the method calls `list_details`, the next method in the CLI flow.

If the choice is not valid, it prints a message to the user to try again, and calls `choose_details` again so the user can enter another choice.

### Validating the user's choice

```ruby
#CLI class
def valid_choice?(input, array)
    input <= array.length && input > 0
end
```

The method `valid_choice?` checks if the input is a non-negative number less than or equal to the length of the `@@detail_categories` array. This is better than hard-coding the allowed inputs - we might want to add more detail categories in the future, and referencing the array directly will reduce the number of updates needed if we do.

It returns `true` if the choice is valid, and `false` if it is not.

## Listing the details

```ruby
#CLI class
def list_details(chosen_details)
    case chosen_details
    when 1
        output_detail_results(@current_word.synonyms, "synonyms")
    when 2
        output_detail_results(@current_word.antonyms, "antonyms")
    when 3
        output_detail_results(@current_word.similar_words, "similar words")
    when 4
        output_detail_results(@current_word.rhymes, "rhymes")
    end
    choose_next_action
end
```

This method accepts the user's input as an argument, and uses it to decide which of the current word's detail attributes should be printed out. It passes the chosen attribute, (along with the string that represents it for printing purposes), to the method `output_detail_results`.

After `output_detail_results` runs, it calls the next method in the CLI flow - `choose_next_action`.

### Printing the detail results

This method prints the chosen details, if any exist.

If the detail array is empty, it displays a message to the user that no details were found.

If the array is not empty, it uses the helper method `print_detail_array` to iterate through the details and print them as an unordered list.

```ruby
#CLI class
def output_detail_results(word_details_array, word_details)
    if word_details_array.empty?
        puts "\nThe word '#{@current_word.word_text}' has no listed #{word_details}.".colorize(:light_blue)
    else
        puts "\nHere are the #{word_details} for '#{@current_word.word_text}':".colorize(:light_blue)
        print_detail_array(word_details_array)
    end
end

def print_detail_array(array)
    array.each do |item|
        puts "- #{item}"
    end
end
```

## Choosing the next action

The `choose_next_action` method allows the user to get more details on the current word (by calling `list_detail_categories`), or look up another word (by calling `choose_word`) without exiting and restarting the program. It also prints a goodbye message if the user chooses to exit.

```ruby
#CLI class
def choose_next_action
    puts "\nWhat would you like to do?".colorize(:light_green)
    puts "\nEnter 1 for more details on '#{@current_word.word_text}'."
    puts "Enter 2 to look up a new word."
    puts "Enter 3 to exit the program."
    next_action = gets.strip.to_i
    case next_action
    when 1
        list_detail_categories
    when 2
        choose_word
    when 3
        puts "\nGoodbye!".colorize(:light_cyan)
        puts "\n"
        exit
    else
        puts "This is not a valid selection. Please try again.".colorize(:light_red)
        choose_next_action
    end
end
```

## The code in action

![Word Lookup Demo](https://dev-to-uploads.s3.amazonaws.com/i/4otc6d6zqtajfc3ob0yd.png)
 
## Thanks for reading :D

If you have any questions or feedback, feel free to find me on Twitter [@DevCait](https://twitter.com/DevCait)!

