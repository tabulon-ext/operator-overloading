### How does it work?

You can only overload certain operators, which are shown in the table. The operators are also listed in the _%overload::ops_ hash made available when you use overload, though the categorization is a little different there.

| __Category__ | __Operators__ |
|---|---|
| Conversion | `""`, `0+`, `bool` |
| Arithmetic | `+`, `-`, `*`, `/`, `%`, `**`, `x`, `.`, `neg` |
| Logical | `!` |
| Bitwise | `&`, `|`, `~`, `^`, `!`, `<<`, `>>` |
| Assignment | `+=`, `-=`, `*=`, `/=`, `%=`, `**=`, `x=`, `.=`, `<<=`, `>>=`, `++`, `--` |
| Comparison | `==`, `<`, `<=`, `>`, `>=`, `!=`, `<=>`, `lt`, `le`, `gt`, `ge`, `eq`, `ne`, `cmp` |
| Mathematical | `atan2`, `cos`, `sin`, `exp`, `abs`, `log`, `sqrt` |
| Iterative | `<>` |
| Dereference | `${}`, `@{}`, `%{}`, `&{}`, `*{}` |
| Pseudo | `nomethod`, `fallback`, `=>` |

Note that `neg`, `bool`, `nomethod`, and `fallback` are not actual Perl operators. The five dereferencers, `""`, and `0+` probably don't _seem_ like operators either. Nevertheless, they are all valid keys for the parameter list you provide to _use overload_. This is not really a problem. We'll let you in on a little secret: it's a bit of a fib to say that the _overload_ pragma overloads operators. It overloads the underlying operations, whether invoked explicitly via their "official" operators, or implicitly via some related operator. (The pseudo-operators we mentioned can only be invoked implicitly.) In other words, overloading happens not at the syntactic level, but at the semantic level. The point is not to look good. The point is to do the right thing. Feel free to generalize.

Note also that `=` does _not_ overload Perl's assignment operator, as you might expect. That would not do the right thing. More on that later.

We'll start by discussing the conversion operators, not because they're the most obvious (they aren't), but because they're the most useful. Many classes overload nothing but stringification, specified by the "" key. (Yes, that really is two double-quotes in a row.)

##### Conversion operators

These three keys let you provide behaviors for Perl's automatic conversions to strings, numbers, and Boolean values, respectively.

We say that _stringification_ occurs when any nonstring variable is used as a string. It's what happens when you convert a variable into a string via printing, interpolation, concatenation, or even by using it as a hash key. Stringification is also why you see something like _SCALAR(0xba5fe0)_ when you try to _print_ an object.

We say that _numification_ occurs when a nonnumeric variable is converted into a number in any numeric context, such as any mathematical expression, array index, or even as an operand of the `..` range operator.

Finally, while nobody here quite has the nerve to call it _boolification_, you can define how an object should be interpreted in a Boolean context (such as `if`, `unless`, `while`, `for`, `and`, `or`, `&&`, `||`, `?:`, or the block of a _grep_ expression) by creating a _bool_ handler.

Any of the three conversion operators can be _autogenerated_ if you have any one of them (we'll explain autogeneration later). Your handlers can return any value you like. Note that if the operation _that_ triggered the conversion is also overloaded, that overloading will occur immediately afterward.

Here's a demonstration of `""` that invokes an object's `as_string` handler upon stringification. Don't forget to quote the quotes:

```perl
package Person;

use overload q("") => \&as_string;

sub new {
    my $class = shift;
    return bless { @_ } => $class;
}

sub as_string {
    my $self = shift;
    my ($key, $value, $result);
    while (($key, $value) = each %$self) {
        $result .= "$key => $value\n";
    }
    return $result;
}

$obj = Person->new(height => 72, weight => 165, eyes => "brown");

print $obj;
```

Instead of something like `Person=HASH(0xba1350)`, this prints (in hash order):

```perl
weight => 165
height => 72
eyes => brown
```

(We sincerely hope this person was not measured in kg and cm.)

##### Arithmetic operators

These should all be familiar except for `neg`, which is a special overloading key for the unary minus: the `-` in `-123`. The distinction between the `neg` and `-` keys allows you to specify different behaviors for unary minus and binary minus, more commonly known as subtraction.

If you overload `-` but not `neg`, and then try to use a unary minus, Perl will emulate a `neg` handler for you. This is known as _autogeneration_, where certain operators can be reasonably deduced from other operators (on the assumption that the overloaded operators will have the same relationships as the regular operators). Since unary minus can be expressed as a function of binary minus (that is, `-123` is equivalent to `0 - 123`), Perl doesn't force you to overload `neg` when `-` will do. (Of course, if you've arbitrarily defined binary minus to divide the second argument by the first, unary minus will be a fine way to throw a divide-by-0 exception.)

Concatenation via the `.` operator can be autogenerated via the stringification handler (see `""` above).

##### Logical operator

If a handler for `!` is not specified, it can be autogenerated using the `bool`, `""`, or `0+` handler. If you overload the `!` operator, the not operator will also trigger whatever behavior you requested. (Remember our little secret?)

You may be surprised at the absence of the other logical operators, but most logical operators can't be overloaded because they short-circuit. They're really control-flow operators that need to be able to delay evaluation of some of their arguments. That's also the reason the `?:` operator isn't overloaded.

##### Bitwise operators

The `~` operator is a unary operator; all the others are binary. Here's how we could overload `>>` to do something like `chop`:

```perl
package ShiftString;

use overload
    '>>' => \&right_shift,
    '""' => sub { ${ $_[0] } };

sub new {
    my $class = shift;
    my $value = shift;
    return bless \$value => $class;
}

sub right_shift {
    my ($x, $y) = @_;
    my $value = $$x;
    substr($value, -$y) = "";
    return bless \$value => ref($x);
}

$camel = ShiftString->new("Camel");
$ram = $camel >> 2;
print $ram;            # Cam
```

##### Assignment operators

These assignment operators might change the value of their arguments or leave them as is. The result is assigned to the lefthand operand only if the new value differs from the old one. This allows the same handler to be used to overload both `+=` and `+`. Although this is permitted, it is seldom recommended, since by the semantics described later under "When an Overload Handler Is Missing (nomethod and fallback)", Perl will invoke the handler for `+` anyway, assuming `+=` hasn't been overloaded directly.

Concatenation (`.=`) can be autogenerated using stringification followed by ordinary string concatenation. The `++` and `--` operators can be autogenerated from `+` and `-` (or `+=` and `-=`).

Handlers implementing `++` and `--` are expected to mutate (alter) their arguments. If you wanted autodecrement to work on letters as well as numbers, you could do that with a handler as follows:

```perl
package MagicDec;

use overload
    q(--) => \&decrement,
    q("") => sub { ${ $_[0] } };

sub new {
    my $class = shift;
    my $value = shift;
    bless \$value => $class;
}

sub decrement {
    my @string = reverse split(//, ${ $_[0] } );
    my $i;
    for ($i = 0; $i < @string; $i++ ) {
        last unless $string[$i] =~ /a/i;
        $string[$i] = chr( ord($string[$i]) + 25 );
    }
    $string[$i] = chr( ord($string[$i]) - 1 );
    my $result = join('', reverse @string);
    $_[0] = bless \$result => ref($_[0]);
}

package main;

for $normal (qw/perl NZ Pa/) {
    $magic = MagicDec->new($normal);
    $magic--;
    print "$normal goes to $magic\n";
}
```

That prints out:

```perl
perl goes to perk
NZ goes to NY
Pa goes to Oz
```

exactly reversing Perl's magical string autoincrement operator.

The `++$a` operation can be autogenerated using `$a += 1` or `$a = $a + 1`, and `$a--` using `$a -= 1` or `$a = $a - 1`. However, this does not trigger the copying behavior that a real `++` operator would. See "The Copy Constructor" later in this chapter.

##### Comparison operators

If `<=>` is overloaded, it can be used to autogenerate behaviors for `<`, `<=`, `>`, `>=`, `==`, and `!=`. Similarly, if `cmp` is overloaded, it can be used to autogenerate behaviors for `lt`, `le`, `gt`, `ge`, `eq`, and `ne`.

Note that overloading `cmp` won't let you sort objects as easily as you'd like, because what will be compared are the stringified versions of the objects instead of the objects themselves. If that was your goal, you'd want to overload `""` as well.

##### Mathematical functions

If `abs` is unavailable, it can be autogenerated from `<` or `<=>` combined with either unary minus or subtraction.

An overloaded `-` can be used to autogenerate missing handlers for unary minus or for the `abs` function, which may also be separately overloaded. (Yes, we know that `abs` looks like a function, whereas unary minus looks like an operator, but they aren't all that different as far as Perl's concerned.)

##### Iterative operator:

The `<>` handler can be triggered by using either readline (when it reads from a filehandle, as in while (`<FH>`)) or `glob` (when it is used for fileglobbing, as in `@files = <*.*>`).

```perl
package LuckyDraw;
 
use overload
    '<>' => sub {
        my $self = shift;
        return splice @$self, rand @$self, 1;
    };
 
 
sub new {
    my $class = shift;
    return bless [@_] => $class;
}
 
package main;
 
$lotto = new LuckyDraw 1 .. 51;
 
for (qw(1st 2nd 3rd 4th 5th 6th)) {
    $lucky_number = <$lotto>;
    print "The $_ lucky number is: $lucky_number.\n";
}
 
$lucky_number = <$lotto>;
print "\nAnd the bonus number is: $lucky_number.\n";
```

In California, this prints:

```perl
The 1st lucky number is: 18
The 2nd lucky number is: 11
The 3rd lucky number is: 40

The 4th lucky number is: 7
The 5th lucky number is: 51
The 6th lucky number is: 33
 
And the bonus number is: 5
```

##### Dereference operators

Attempts to dereference scalar, array, hash, subroutine, and glob references can be intercepted by overloading these five symbols.

The online Perl documentation for overload demonstrates how you can use this operator to simulate your own pseudohashes. Here's a simpler example that implements an object as an anonymous array but permits hash referencing. Don't try to treat it as a real hash; you won't be able to _delete_ key/value pairs from the object. If you want to combine array and hash notations, use a real pseudohash (as it were).

```perl
package PsychoHash;

use overload '%{}' => \&as_hash;

sub as_hash {
    my ($x) = shift;
    return { @$x };
}

sub new {
    my $class = shift;
    return bless [ @_ ] => $class;
}

$critter = new PsychoHash( height => 72, weight => 365, type => "camel" );

print $critter->{weight};   # prints 365
```

Also see [Chapter 14, "Tied Variables"](http://docstore.mik.ua/orelly/perl3/prog/ch14_01.htm), for a mechanism to let you redefine basic operations on hashes, arrays, and scalars.

<hr>

When overloading an operator, try not to create objects with references to themselves. For instance,

```perl
use overload '+' => sub { bless [ \$_[0], \$_[1] ] };
```

This is asking for trouble, since if you say `$animal += $vegetable`, the result will make `$animal` a reference to a blessed array reference whose first element is `$animal`. This is a circular reference, which means that even if you destroy `$animal`, its memory won't be freed until your process (or interpreter) terminates. See "Garbage Collection, Circular References, and Weak References" in [Chapter 8](http://docstore.mik.ua/orelly/perl3/prog/ch08_01.htm), "References".

### Example

```perl
# cpanm Carp::Assert
use Carp::Assert;

{
  package vector;
  use overload
    '+' => "plus",
    '-' => "minus";

  sub new {
    my ($class, $x, $y) = @_;
    my %info;
    $info{x} = $x;
    $info{y} = $y;
    bless \%info, $class;
  }

  sub area {
    my ($self) = @_;
    $$self{x} * $$self{y}
  }

  # Overrides + (a + b).
  sub plus {
    my ($this, $that) = @_;
    new vector($$this{x} + $$that{x}, $$this{y} + $$that{y});
  }

  # Overrides - (a - b).
  sub minus {
    my ($this, $that) = @_;
    new vector($$this{x} - $$that{x}, $$this{y} - $$that{y});
  }
}

$v1 = new vector(3,6);
$v2 = new vector(5,8);

assert(($v1 + $v2)->{x} == 8 and ($v1 + $v2)->{y} == 14);
assert(($v2 - $v1)->{x} == 2 and ($v2 - $v1)->{y} == 2);
```

### Further reading

- [Overloadable Operators (Programmin Perl)](http://docstore.mik.ua/orelly/perl3/prog/ch13_03.htm)
- [Operator Overloading in Perl](http://www.wellho.net/resources/ex.php4?item=p218/opolop)