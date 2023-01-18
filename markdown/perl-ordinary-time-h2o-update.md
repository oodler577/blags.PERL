During the 2022 [Perl Advent](https://perladvent.org/2022/2022-12-06.html), in particular the entry for [December 06](https://perladvent.org/2022/2022-12-06.html); we were introduced to a module called `Util::H2O`.

A lot has already been said about `Util::H2O`, and this author (Oodler, Mayor of _Flavortown_) uses it a lot in client and production code. So much so, that he created the `Util::H2O::More` module to encapsulate some common tasks and additional capabilities for working between _pure_ Perl data structures and _blessed_ objects that have real data _accessors_.

As a result of the Perl Advent post, another need was revealed. The original intent of `h2o` was only to take a purely `HASH` reference based data structure, and with this intent it shall return (as it should). And even though `h2o` supports the `-recurse` option; it refuses to enter into anything but a `HASH` ref. That means if it encounters the other legitimate container reference, `HTML`. This restriction applies similarly to `o2h`, the inverse (or _undoer_) of `h2o`.

A new _keyword_ (because this is how they should be viewed) was introduced in `Util::H2O::More`, called `d2o` and it's _undoer_, `o2d`. Essentially, it traverses a Perl data structure, looking for pure `HASH` refs that may be potentially contained inside of `ARRAY` refs.

Below is the `JSON` from the Perl Advent entry on December 06, 2022. In this entry, `h2o` was used to _objectify_ each individual user record since when _decoded_ from the `JSON`, it became a `HASH` ref. Because `h2o` only works with data structures that contain purely `HASH` references, an explicit iteration step had to be introduced to apply `h2o` to each `HASH` ref. That is,

\# decode JSON from response content
my $json\_array\_ref = JSON::decode\_json($response->content); # $json is an ARRAY reference

foreach my $person (@$json\_array\_ref) {

    # -recurse creates deep accessors, e.g.,
    #  $person->address->geo->lat;

    h2o -recurse, $person;

    ...

}

Was used to _objectify_ each record contained in `JSON` of the form,

\[
  {
    "id": 1,
    "name": "Leanne Graham",
    "username": "Bret",
    "email": "Sincere@april.biz",
    "address": {
      "street": "Kulas Light",
      "suite": "Apt. 556",
      "city": "Gwenborough",
      "zipcode": "92998-3874",
      "geo": {
        "lat": "-37.3159",
        "lng": "81.1496"
      }
    }
  },
 ...
\]

The problem revealed here is that `h2o` is amazing for giving accessors those ad hoc or temporary _pure_ `HASH` references all Perl programmers come to know and love. But the veil gets pierced once an `ARRAY` ref is encountered. And this is what happens in the example above.

How should this be solved? By generalizing `h2o` into a _keyword_ that has the ability to traverse any Perl data structure, adding accessors to any `HASH` ref it finds along this way. This is what `d2o` does.

Now the explicit iteration that is just for adding accessors to each record, goes from a loop having to dereference the ``` ARRAY ``ref in `$json_array_ref`, to the following:`` ```

    # decode JSON from response content
    my $json_array_ref = d2o JSON::decode_json($response->content); # $json is an ARRAY reference
    

``` `` `d2o` does not just ignore `ARRAYs`, it goes a step further by providing _virtual_ methods around the `ARRAY` container that further helps avoid the need to use `ARRAY`dereferencing syntax to iterate over the `HASH` references (now blessed with accessors). See the example below,  foreach my $user ($json_array_ref->all) {   printf qq{%5.4f, %5.4f, %s, %s\n},     $person->address->geo->lat,   # deep chain of accessors from '-recurse'     $person->address->geo->lng,   # deep chain of accessors from '-recurse'     $person->name,     $person->username;   }  Or more concisely,  use strict; use warnings; use Util::H2O::More qw/d2o/;  my $http     = HTTP::Tiny->new; my $response = h2o $http->get(q{https://jsonplaceholder.typicode.com/users});  # decode JSON from response content my $json_array_ref = d2o JSON::decode_json( $response->content );    # $json is an ARRAY reference  foreach my $user ( $json_array_ref->all ) {     printf qq{%5.4f, %5.4f, %s, %s\n}, $person->address->geo->lat, $person->address->geo->lng, $person->name, $person->username; } `` ```
