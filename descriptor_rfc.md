# Descriptor RFC
## Intro

This RFC describes a rough outline of how I imagine the descriptor project
functioning inside the existing `cargo test` ecosystem. We'll outline some of
the motivations and specific implementation details. Feedback is highly
encouraged in the form of comments on the PR to this document.

I imagine the descriptor project leaning heavily on inspiration from RSpec.
RSpec is the 'spec style' testing framework that I help maintain in Ruby, and is
the originating system for a number of these ideas.

## Motivations

The testing ecosystem in Rust with Cargo defines an API that allows users to
express tests via fairly standard "xunit" style of writing tests. Of note, there
are currently no setup or teardown style hooks. Additionally, as advised in the
[rust book](https://doc.rust-lang.org/book/testing.html), users typically mix
testing code between to their library modules, and a testing directory.

## DRY testing

In [RSpec](http://rspec.info/documentation/3.5/rspec-core/) (specifically RSpec
core) our testing system is designed to support "context" based testing. The
idea here is to allow a significant reduction of duplication between tests via
the use of setup constructs (e.g. `before`, `let`) and permuting what happens in
each of those contexts. Let's look at the following Ruby code:

```ruby
require "rspec/autorun"

class Recipie
  def initialize(ingredient_1, ingredient_2)
    @ingredient_1 = ingredient_1
    @ingredient_2 = ingredient_2
  end

  def call
    if @ingredient_1 == "fresh"
      @ingredient_1.upcase!
      @ingredient_1 + @ingredient_2
    else
      @ingredient_2
    end
  end
end

RSpec.describe Recipie do
  subject(:recipie) { Recipie.new(ingredient_1, ingredient_2) }
  let(:ingredient_2) { "carrot" }

  context "ingredient 1 is fresh" do
    let(:ingredient_1) { "fresh" }

    it "upcases ingredient 1" do
      expect(ingredient_1).to receive(:upcase!).and_return("FRESH")

      recipie.call
    end

    it "returns the combined ingredients" do
      expect(recipie.call).to eq("FRESHcarrot")
    end
  end

  context "ingredient 2 is not fresh" do
    let(:ingredient_1) { "poop" }

    it "returns the second ingrident only" do
      expect(recipie.call).to eq(ingredient_2)
    end
  end
end

```

In this example we see the usage of `let` to construct test setup that does not
have to be duplicated in individual tests, allowing users to add test cases for
sides of conditionals in the unit without having to repeat the same setup. This
DRY "context based" testing is one of the core ideas behind RSpec.

### Docstrings/generative tests

The other core motivation here is, that in RSpec, arbitrary strings are valid as
test descriptions, and test descriptions are method calls. The arbitrary
docstrings allow users to provide rich description (or specification, if you
will) of their program. These nested descriptions allow for human readable
"documentation" output to be generated, for example: ￼

In addition to this, tests being method calls with strings as their descriptions
allow tests to be generated programmatically, which can help with randomised and
or iterative generation of tests.

## Examples

I imagine the descriptor API will function as a compiler plugin that users put
in their test files (or inline in lib files). All descriptor tests will be
encapsulated inside a `descriptor!` block which will encapsulate the
environment, generate a data structure, and then output tests which can be
understood by cargo test. Without going into too much detail, here are some
examples of what I imagine descriptor test files looking like

### a simple descriptor test

```rust
fn add_two(n: i32) -> i32 {
    n + 2
}

descriptor!(
    describe("fn add_two", || {
        it("adds two to a passed number", || {
            assert_eq!(add_two(3), 5);
        });
    });
)
```

### A more complex descriptor test

```rust

fn add_two(n: i32) -> i32 {
    n + 2
}

fn add_three(n: i32) -> i32 {
    n + 3
}

struct Cat {
    pub tail_length: i32,
}

descriptor!(
    describe("adding numbers", || {
        subject(base, || { 3 } );

        describe("add_two", || {
            bind(expected, || {
                let c = Cat { tail_length: 1 };
                c.tail_length += 1;
                c
            });

            before(|| {
                expected.tail_length += 3;
            });

            it("adds two to a passed number", || {
                assert_eq!(add_two(base), expected.tail_length);
                expected.tail_length = 17;
            });

            it("resets binds between calls", || {
                assert!(expected.tail_length != 17);
            });
        });

        describe("add_three", || {
            it("adds three to a passed number", || {
                assert_eq!(add_three(base), 6);
            });

            context("a really big number", || {
                subject(base, || { 1337 });

                it("gets the new subject", || {
                    assert_eq(base, 1337);
                });
            });
        });
    });
)
```

## Specific explanations of feature

### Example Groups (`describe` and `context`)

**Example Groups** are the base unit of test structuring in descriptor. While
not tests themselves they contain setup for those tests and are the places from
which we hang tests. **Example Groups** can have arbitrarily many child
**Example Groups** nested under them. Child **Example Groups** inherit all the
setup behaviour of their parents, and can also override those behaviours as
necessary. Child **Example Groups** do not inherit the **Examples** (`it`s) from
their parents however.

We can put arbitrarily many **Examples** inside an example group, by calling
`it`.  The behaviour in RSpec is that Child **Example Groups** get executed in
an essentially random order. In Rust, it'd nice for them to be executed in
parallel by default, given that it's easy to make that safe

### Examples (`it`)

Examples are what get translated into tests in descriptor. **Examples** cannot
live at the top level but must be nested under an **Example Group**. The
**description** of each example is the concatenated strings of all its parent
example groups up to the root, plus its own string description. I imagine that
via the use of the compiler plugin system each it will be output into a test, so
for example:

```rust
descriptor!(
    describe("group", || {
        it("example", || {
           assert!(true);
        });
    });
)
```

Would get translated via the compiler plugin to

```rust
#[test]
fn descriptor_1_1_group_example() {
  assert!(true);
}
```

A future API that would be very useful here would be the ability to provide an
arbitrary data structure that [some hypothetical
formatter](#formatter_discussion) could consume to the test. e.g:

```rust
#[test(description=Descriptor::ExampleDescription { parents: vec!("group"), description: "example" } ]
fn descriptor_1_1_group_example() {
  assert!(true);
}
```

The numbers in the test string here represent the indices of the individual
**Example Groups** and **Examples** as they are encountered in the file. The
last number is always an **Example**, and there can be arbitrarily many
**Example Group** numbers in front. e.g. nesting two describes and a context,
then an it would produce `descriptor_1_1_1_1`. Having a second context under two
describes would produce `descriptor_1_1_2_1` and so on.

### Execution hooks

Execution hooks provide part of the setup and teardown story for descriptor
tests. They enable the user to provide a per **Example Group** configuration for
what should happen in that test. For example we might have a test that interacts
with the database via the [diesel](https://github.com/diesel-rs/diesel) crate.

```rust
descriptor!(
  describe("databasing", || {
    before(|| {
      let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
      let connection = PgConnection::establish(&database_url).expect(&format!("Error connecting to {}", database_url))
    });

    after(|| {
      diesel::delete(posts.all).execute(connection).expect();
    });

    it("can use the database connection", || {
      let new_post = NewPost {
        title: title,
        body: body,
      };

      diesel::insert(&new_post).into(posts::table)
        .get_result(conn)
        .expect("Error saving new post")
    });
  });
)

```

Would get translated into

```rust
#[test]
fn descriptor_1_1_databasing_can_use_the_database_connection() {
  let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
  let connection = PgConnection::establish(&database_url).expect(&format!("Error connecting to {}", database_url))

  let new_post = NewPost {
    title: title,
    body: body,
  };

  diesel::insert(&new_post).into(posts::table)
    .get_result(conn)
    .expect("Error saving new post")

  diesel::delete(posts.all).execute(connection).expect();
}

What's really compelling here, is not the story for a single level of nesting,
but when we create multiple levels of nesting to allow users to permute how they
write their tests.

```rust
descriptor!(
  describe("databasing", || {
    before(|| {
      let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
      let connection = PgConnection::establish(&database_url).expect(&format!("Error connecting to {}", database_url))
    });

    after(|| {
      diesel::delete(posts.all).execute(connection).expect();
    });

    it("can use the database connection", || {
      let new_post = NewPost {
        title: "a title",
        body: "a body",
      };

      diesel::insert(&new_post).into(posts::table)
        .get_result(conn)
        .expect("Error saving new post")
    });

    context("with a post in the database", || {
      before(|| {
          let new_post = NewPost {
            title: "a different title",
            body: "a different body",
          };

          diesel::insert(&new_post).into(posts::table)
            .get_result(conn)
            .expect("Error saving new post")
          });
      });

      it("can insert another post", || {
          let new_post = NewPost {
            title: "a third title",
            body: "a third body",
          };

          diesel::insert(&new_post).into(posts::table)
            .get_result(conn)
            .expect("Error saving new post")
          });
      });
    });
  });
)
```

Gets translated into

```rust
#[test]
fn descriptor_1_1_databasing_can_use_the_database_connection() {
  let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
  let connection = PgConnection::establish(&database_url).expect(&format!("Error connecting to {}", database_url))

  let new_post = NewPost {
    title: title,
    body: body,
  };

  diesel::insert(&new_post).into(posts::table)
    .get_result(conn)
    .expect("Error saving new post")

  diesel::delete(posts.all).execute(connection).expect();
}

#[test]
fn descriptor_1_1_1_databasing_with_a_post_in_the_database_can_insert_another_post {
  // outer before
  let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
  let connection = PgConnection::establish(&database_url).expect(&format!("Error connecting to {}", database_url))

  // inner before
  let new_post = NewPost {
    title: "a different title",
    body: "a different body",
  };

  diesel::insert(&new_post).into(posts::table)
    .get_result(conn)
    .expect("Error saving new post")

  // inner example
  let new_post = NewPost {
    title: "a third title",
    body: "a third body",
  };

  diesel::insert(&new_post).into(posts::table)
    .get_result(conn)
    .expect("Error saving new post")
  });

  // outer after (there is no inner after)
  diesel::delete(posts.all).execute(connection).expect();
}
```

### Binds

`bind`s in descriptor are like `lets` in RSpec, they are given a lambda which
gets evaluated into an accessible local for the duration of an **example**. In
this design, binds are essentailly shorthand for a let in a `before` block.

## Formatters Discussion
## Open questions

* Is this acheivable with compiler plugins today?

* Ruby users, is there anything from RSpec I missed that is desperately needed
  or that would require extra hooks in cargo?

* What of this is not acheivable in cargo today?

* Is there a better design for binds such that they can be made lazy and memoized
  like they do in RSpec?
