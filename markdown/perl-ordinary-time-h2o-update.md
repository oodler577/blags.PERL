# Background

During the 2022 [Perl Advent](https://perladvent.org/2022/2022-12-06.html), in particular the entry for [December 06](https://perladvent.org/2022/2022-12-06.html); we were introduced to a module called `Util::H2O`.

A lot has already been said about `Util::H2O`, and this author (Oodler, _Mayor of Flavortown_) uses it a lot in client and production code. So much so, that he created the `Util::H2O::More` module to encapsulate some common tasks and additional capabilities for working between _pure_ Perl data structures and _blessed_ objects that have real data _accessors_.

# Pure `HASH` Data Structures Only?

`h2o` is perfect for dealing with data structures that are made up of strictly `HASH` references, but it is often the case that useful data structures contain a mix of `HASH` and `ARRAY` references. For example, when using databases or web API calls for `JSON`, it is often a list of records that is returned. This was the case of the example call that was in the original Perl Advent 2022 article. 

Recall the original example,

```
use strict;
use warnings;
use JSON       qw//;
use HTTP::Tiny qw//;
use Util::H2O;    # only exports 'h2o'

# give's Santa "$response->content", "$response->status", "$response->success", etc
# from HTTP::Tiny's response object (pure HASH)

my $http     = HTTP::Tiny->new;
my $response = h2o $http->get(q{https://jsonplaceholder.typicode.com/users});

# check for unsuccessful web request

if ( not $response->success ) {
    print STDERR qq{Can't get list of online persons to watch!\n};
    printf STDERR qq{Web request responded with with HTTP status: %d\n}, $response->status;
    exit 1;
}

# decode JSON from response content
my $json_array_ref = JSON::decode_json( $response->content );    # $json is an ARRAY reference

print qq{lat, lng, name, username\n};

foreach my $person (@$json_array_ref) {

    # -recurse creates deep accessors, e.g.,
    #  $person->address->geo->lat;

    h2o -recurse, $person;

    printf qq{%5.4f, %5.4f, %s, %s\n}, $person->address->geo->lat,    # deep chain of accessors from '-recurse'
      $person->address->geo->lng,                                     # deep chain of accessors from '-recurse'
      $person->name, $person->username;
}
```

Which outputs:

```
lat, lng, name, username
-37.3159, 81.1496, Leanne Graham, Bret
-43.9509, -34.4618, Ervin Howell, Antonette
-68.6102, -47.0653, Clementine Bauch, Samantha
29.4572, -164.2990, Patricia Lebsack, Karianne
-31.8129, 62.5342, Chelsey Dietrich, Kamren
-71.4197, 71.7478, Mrs. Dennis Schulist, Leopoldo_Corkery
24.8918, 21.8984, Kurtis Weissnat, Elwyn.Skiles
-14.3990, -120.7677, Nicholas Runolfsdottir V, Maxime_Nienow
24.6463, -168.8889, Glenna Reichert, Delphine
-38.2386, 57.2232, Clementina DuBuque, Moriah.Stanton
```

# Some New Keywords

A new _keyword_ has been introduced in `Util::H2O::More`, called `d2o` and it's _undoer_, `o2d` (like `o2h` is to `h2o`). Essentially, it traverses a Perl data structure, looking for pure `HASH` refs that may be potentially contained inside of `ARRAY` refs.

Without much ado:

```
use strict;
use warnings;
use Util::H2O::More qw/h2o d2o/;
my $http     = HTTP::Tiny->new;
my $response = h2o $http->get(q{https://jsonplaceholder.typicode.com/users});  # decode JSON from response content
my $json_array_ref = d2o JSON::decode_json( $response->content );
# $json is an ARRAY reference
foreach my $user ( $json_array_ref->all ) {
  printf qq{%5.4f, %5.4f, %s, %s\n},
  $person->address->geo->lat,
  $person->address->geo->lng,
  $person->name,
  $person->username;
}
```

In this example, the initial `HASH` reference returned by `HTTP::Tiny` was made into an object with accessors using `h2o` like the original code. However, rather than having to dereference the `ARRAY` reference based data structure returned after decoding by `decode_json`, `d2o` is employed to convert the data structure such that all `HASH` references haveaccessors as expected. And the `ARRAY` reference containing the list of `HASH` references has been _blessed_ so that it has the _virtual_ methods briefly mentioned above.

This allows the code to collapse from:

```
foreach my $person (@$json_array_ref) {
    h2o -recurse, $person;
...
```
to:

`foreach my $user ( $json_array_ref->all ) {`
