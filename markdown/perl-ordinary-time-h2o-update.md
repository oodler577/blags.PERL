# Background

During the 2022 [Perl Advent](https://perladvent.org/2022/2022-12-06.html), in particular the entry for [December 06](https://perladvent.org/2022/2022-12-06.html); the wortld was introduced to a module called `Util::H2O`.

A lot has already been said about `Util::H2O`, and this author (Oodler, _Mayor of Flavortown_) uses it a lot in client and production code. So much so, that he created the `Util::H2O::More` module to encapsulate some common tasks and additional capabilities for working between _pure_ Perl data structures and _blessed_ objects that have real data _accessors_, in a natural and idiomatic way.

# Support for Generic Perl Data Structures

`h2o` is perfect for dealing with data structures that are made up of strictly `HASH` references, but it is often the case that useful data structures contain a mix of `HASH` and `ARRAY` references. For example, when using databases or web API calls returning `JSON`, it is often a list of records that is returned. This was the case of the example call that was in the _December 06_ Perl Advent 2022 article. 

Recall the original example,

    use strict;
    use warnings;
    use JSON       qw//;
    use HTTP::Tiny qw//;
    use Util::H2O;    # only exports 'h2o'
    my $http     = HTTP::Tiny->new;
    my $response = h2o $http->get(q{https://jsonplaceholder.typicode.com/users});
    if ( not $response->success ) {
        print STDERR qq{Cannot get list of online persons to watch!\n};
        printf STDERR qq{Web request responded with with HTTP status: %d\n}, $response->status;
        exit 1;
    }
    my $json_array_ref = JSON::decode_json( $response->content );

    print qq{lat, lng, name, username\n};
    foreach my $person (@$json_array_ref) {

        # objectify each HASH reference at a time
        h2o -recurse, $person;
        printf qq{%5.4f, %5.4f, %s, %s\n},
          $person->address->geo->lat,
          $person->address->geo->lng,
          $person->name, $person->username;
    }

Which outputs:

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

# Some New Keywords

New keywords have been introduced in `Util::H2O::More`, called `d2o` and it's _undoer_, `o2d` (like `o2h` is to `h2o`). Essentially, it traverses a Perl data structure, looking for pure `HASH` refs that may be potentially contained inside of `ARRAY` refs, and adds accessors. For `ARRAY`s, it adds some handy vmethods to make accessing the lists much cleaner (e.g., `all`).

Without much ado:

    use strict;
    use warnings;
    use JSON            qw//;
    use HTTP::Tiny      qw//;
    use Util::H2O::More qw/h2o d2o/;
    my $http           = HTTP::Tiny->new;
    my $response       = h2o $http->get(q{https://jsonplaceholder.typicode.com/users});
    if ( not $response->success ) {
        print STDERR qq{Cannot get list of online persons to watch!\n};
        printf STDERR qq{Web request responded with with HTTP status: %d\n}, $response->status;
        exit 1;
    }

    # objectify contents of $json_array_ref in one pass
    my $json_array_ref = d2o JSON::decode_json( $response->content );
    
    # $json is an ARRAY reference
    foreach my $person ( $json_array_ref->all ) {
        printf qq{%5.4f, %5.4f, %s, %s\n},
          $person->address->geo->lat,
          $person->address->geo->lng,
          $person->name, $person->username;
    }

Which outputs, like above:

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

In this example, the initial `HASH` reference returned by `HTTP::Tiny` is made into an object with accessors using `h2o` like the original code. However, rather than having to dereference the `ARRAY` reference `$json_array_ref` (returned after decoding by `decode_json`), `d2o` is employed to convert the data structure such that all `HASH` references haveaccessors as expected. And the `ARRAY` reference containing the list of `HASH` references has been _blessed_ so that it has the _virtual_ methods on `ARRAY`s briefly mentioned above.

This allows the code to collapse from:

    foreach my $person (@$json_array_ref) {
        h2o -recurse, $person;
    ...

to, simply:

    foreach my $user ( $json_array_ref->all ) {

thus avoiding the call to `h2o` since the `d2o` rooted out all the `HASH` references buried in `$json_array_ref` and applied `h2o` to them.

More importantly, this requires no _a priori_ knowledge of the data structure ahead of time.

# Conclusion

It is critical to point out, that `h2o` does _one thing_ and does it _very_ well - which is a cherished and time tested UNIX _ideal_ for standard tooling. But as a result of _present_'ing `Util::H2O` in 2022's Perl Advent, it was clear that for this common case, `h2o` by itself was insufficient to idiomatically improve the handling of Perl data structures that was resulting from the web API call and subsequent `decode_json`. The solution is to just add some support for data structure traversal (applying `h2o` along the way), which is all `d2o` does.

So `d2o` was then added to `Util::H2O::More` to make things a little nicer, and thus taking Perl another step closer to a situation that alls programmers to more cleanly work with complex data structures - by eliminating the glut of _curley_ and _square_ braces and the need to dereference data structures along the way. The addition of the virtual methods to the `ARRAY` containers, is just more sweetness.

Please checkout [Util::H2O](https://metacpan.org/pod/Util::H2O) and [Util::H2O::More](https://metacpan.org/pod/Util::H2O::More); alone or combined, the options available to deal with _ad hoc_ or _in flight_ complex data structures in Perl in very clean and idiomatic ways _without_ resorting to so called "POOP" (_Perl Object Oriented Programming_) is continuing to improve. Practical sources of these data structures include, `DBI` (e.g., `DBD::mysql`), `JSON::decode_json`, and `Web::Scraper`. `Util::H2O::More` even provides special methods for working with `Getopt::Long` and `Config::Tiny`, `opt2h2o` and `ini2h2o`, respectively.
