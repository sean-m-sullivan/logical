# Ansible Collection - utoddl.logical - test

The *logical filter* adds some "`if`" / "`elif`" / "`else`" logical capabilities to your Ansible data.
(Also "`and`", "`or`", "`xor`", "`not`", and data promotions; see below.)

Usage: `"{{ data | logical }}"`

## Examples:
### if / elif / else

The dict keys "`if`", "`elif`", and "`else`" must be the only dict keys at their level. They
must also be list elements – i.e. "`- if:`" rather than simply "`if:`" – because order
matters. You can have arbitrarily many sets of `if [ elif ... ][ else ]` in a list.

Each "`if`" and "`elif`" (though not "`else`") must contain a list, the first element of which
is a boolean. These initial booleans are removed as the "`if`" and "`elif`" are evaluated,
and the remaining elements of the "winning" "`if`", "`elif`", or "`else`" replace the "`if`" in
its list. Example:

```yaml
---
- name: demo01
  hosts: localhost
  gather_facts: false
  vars:
    snowflake_1: "{{ [ True, False] | random }}"
    snowflake_2: "{{  [ True, False] | random }}"
    data_before:
      my_color:
       - if:
         - "{{ snowflake_1 }}"
         - purple:
           - the new green
           - tastes like chicken
       - elif:
         - "{{ snowflake_2 }}"
         - Purple Rain
       - else:
           Green
    data_after: "{{ data_before | logical }}"
  tasks:
    - name: A list
      copy:
        content: "{{ {'data_after': data_after} |
                     to_nice_yaml(indent=2, width=1337) }}"
        dest: "/tmp/{{ ansible_play_name }}.txt"
```

The resulting `/tmp/{{ ansible_play_name }}.txt` after subsequent runs of the playbook
above may contain:

```yaml
data_after:
  my_color:
  - purple:
    - the new green
    - tastes like chicken
```
or
```yaml
data_after:
  my_color:
  - Purple Rain
```
or 
```yaml
data_after:
  my_color:
  - Green
```

### and / or / xor

The "`and`", "`or`", and "`xor`" dict keys must be the lone keys in their respective
dicts. They replace themselves with their subordinate contents after evaluation. The
immediate subordinates of "`and`", "`or`", and "`xor`" can be a flat
or nested list of booleans. Subordinate lists are evaluated first, then the corresponding
operator is evaluated and replaced by the resulting boolean.

- "`and`" is true if all its immediate subordinates are true.
- "`or`" is true if any of its immediate subordinates is true.
- "`xor`" is true if exactly one of its immediate subordinates is true and any others is false.

They do not have to appear in the context of an "`if`"/"`elif`"/"`else`" construct.

```yaml
data_before:
    - if:
      - and:
        - true
        - or:
          - true
          - false
        - xor:
          - true
          - false
          - false
        - not: false   # See below.
      - Truth: is beauty
    - else:
        This should not happen
data_after: "{{ data_before | logical }}"
```
results in
```yaml	
data_after:
- Truth: is beauty
```

### not
Just like "`if`", "`elif`", "`else`", "`and`", "`or`", and "`xor`", "`not`" must be a lone dict key.
However, the "`not`" dict key does not require being in a list. It flips the values of all
subordinated booleans, then removes itself from the dict, effectively promoting up one
level all the data below it.

```yaml
data_before:
   alpha:
     not:
      beta:
      - epsilon
      - True
      gamma:
      - zeta
      - - True
        - False
      delta:
      - True
      - False
      - True
```

after passing through the logical filter becomes

```yaml	
data_after:
  alpha:
    beta:
    - epsilon
    - false
    gamma:
    - zeta
    - - false
      - true
    delta:
    - false
    - true
    - false
```

## The "<<" Promotion Prefix Operator

Because "`if`" and friends need to be list elements (to preserve order), and because their
evaluated subordinate data replace the "`if`" in the resulting data, it follows that they can
only produce lists. But sometimes you want to produce dicts. The "`<<`" promotion prefix
addresses this issue.

Consider the following playbook and its output.
```yaml
---
- name: goldilocks
  hosts: localhost
  gather_facts: False
  vars:
    basename: goldilocks
    soup:  "{{ ['too hot', 'too cold','just right'] | random() }}"
    chair: "{{ ['too hard','too soft','just right'] | random() }}"
    bed:   "{{ ['too hard','too soft','just right' ] | random() }}"  # That extra space is necessary!
    data:
      soup:
        is: "{{ soup }}"
        owner:
        - if: ["{{ soup == 'too hot' }}","Papa Bear"]
        - elif: ["{{ soup == 'too cold' }}","Mama Bear"]
        - else: "Baby Bear"
      bed:
        is: "{{ bed }}"
        owner:
        - if: ["{{ bed == 'too hard' }}","Papa Bear"]
        - elif: ["{{ bed == 'too soft' }}","Mama Bear"]
        - else: "Baby Bear"
      chair:
        is: "{{ chair }}"
        owner:
        - if: ["{{ chair == 'too hard' }}","Papa Bear"]
        - elif: ["{{ chair == 'too soft' }}","Mama Bear"]
        - else: "Baby Bear"
    home: "{{ data | logical }}"
  tasks:
  - name: The Bear's Home
    copy:
      content: |
        {{ { 'before' : data } | to_nice_yaml(indent=2, width=1337) }}
        {{ { 'after' : home } | to_nice_yaml(indent=2, width=1337) }}
      dest: /tmp/{{basename}}.txt
```
	
What We Get:
```yaml
after:
  bed:
    is: too hard
    owner:
    - Papa Bear
  chair:
    is: too hard
    owner:
    - Papa Bear
  soup:
    is: too cold
    owner:
    - Mama Bear
```

After applying the logical filter, the "`owner`"s will have become single
element lists. We'd like to promote the "`owner`" values up a level.

What We Want:
```yaml
after:
  bed:
    is: too hard
    owner: Papa Bear
  chair:
    is: too hard
    owner: Papa Bear
  soup:
    is: too cold
    owner: Mama Bear
```

We can do this by adding some "`<<`" promotion prefixes to selected dict keys.
A promotion prefix is "`<<`" followed by an optional modifier:

* "`<<`" &mdash; (no modifier) pre-existing matching keys are replaced by promoted keys.
* "`<<|`" &mdash; pre-existing and promoted matching keys are deep merged, lists are joined and flattened.
* "`<<-`" &mdash; pre-existing and promoted matching keys are deep merged, lists are joined, flattened, and uniqified.

Here's our example again with promotion prefixes applied to the "`owner`" keys:

```yaml
data:
  soup:
    is: "{{ soup }}"
    <<owner:
    - if: ["{{ soup == 'too hot' }}","Papa Bear"]
    - elif: ["{{ soup == 'too cold' }}","Mama Bear"]
    - else: "Baby Bear"
  bed:
    is: "{{ bed }}"
    <<owner:
    - if: ["{{ bed == 'too hard' }}","Papa Bear"]
    - elif: ["{{ bed == 'too soft' }}","Mama Bear"]
    - else: "Baby Bear"
  chair:
    is: "{{ chair }}"
    <<owner:
    - if: ["{{ chair == 'too hard' }}","Papa Bear"]
    - elif: ["{{ chair == 'too soft' }}","Mama Bear"]
    - else: "Baby Bear"
home: "{{ data | logical }}"
```

With the desired result:

```yaml	
after:
  bed:
    is: too hard
    owner: Papa Bear
  chair:
    is: too hard
    owner: Papa Bear
  soup:
    is: just right
    owner: Baby Bear
```

Exactly how promotion works is subtle. Order matters, and understanding that order can avoid surprises.

* The "`<<promotion`" key is removed from its containing dict, and its contents ("`tmp`"
  below) are held for examination.

  | input | output | workspace |
  | :----- | :------ | :--------- |
  | `a: alpha`        | `  `  |  `  `  |
  | ~~`<<b: (whatever)`~~ | ` →  `  |  `  tmp: (whatever)` |
  | `c: charlie`      | `  `  |  `  `  |
  
* If "`tmp`" is not a list, make it one.:

  |  `tmp: ...   →    tmp: [...]` |

* Examine each of the elements of the "`tmp`" list. If the examined element is a dict,
  each of its top-level keys and values move up to the same level as the original
  "`<<promotion`" key. NOTE: if an identically named key already exists at that level,
  the subordinate key and its value either (1) replaces the pre-existing key at the
  "`<<promotion`" key's original level (in the case of "`<<`"), or (2) the subordinate
  data are merged (in the case of "`<<|`" or "`<<-`") with the data of the pre-existing
  key.

  | input | output | workspace |
  | :----- | :------ | :--------- |
  | `a: alpha`    | `a: alpha` |    |
  | ~~`<<b:`~~    | `  `   |           |
  | `  - b1: bar` | `b1: bar` |  `←  tmp: { b1: bar }` |
  | `b1: goner`   | ~~`b1: goner`~~  |    |
  | `c: charlie`  | `c: charlie` |    |
  
* If there are any elements left in the "`tmp`" list, (i.e. elements which weren't
  promoted dicts), and the "`<<promotion`" key had any characters to the right of "`<<`",
  then "`tmp`" is added back as a value for the "`<<promotion`" key at the same level as
  the original "`<<promotion`" key and with the same key name but with the "`<<`"
  removed. If an identically named key already exists at that level, then the promotion
  key and its value either (1) replace that pre-existing key at the "`<<promotion`" key's
  original level (in the "<<" case), or (2) the subordinate data are merged (in the "<<|"
  or "<<-"cases ) with that of the pre-existing key. If only one element remains in the
  list, then that element becomes the value for the new key rather than the single
  element list.

  | input | output | workspace |
  | :----- | :------ | :--------- |
  |  `a: alpha`         |    `a: alpha`       |                        |
  |  `<<b:`             |    `b: keeper`      |  ` ← tmp: [ keeper ]`  |
  |  `  - keeper`       |    `  `             |                        |
  |  `  - b1: bar`      |    `b1: bar`        |  ` ← tmp: { b1: bar }` |
  |  `b1: goner`        |    ~~`b1: goner`~~  |                        |
  |  `<<c:`             |    `c:`             |                        |
  |  `  - super`        |    `  - super`      |                        |
  |  `  - supper`       |    `  - supper`     |                        |
  |  `  - c1: pepper`   |    `c1: pepper`     |  ` ← tmp: { c1: pepper }` |
  |  `d: charlie`       |    `d: charlie`     |                        |


The result of an `- if:` is always a list, even if it has a single element. If that
single element is a dict that you want in place of the list produced by the `- if:`, you
may have to insert a sacrificial dummy "`<<promotion`" key above the `- if:` block.
You may also use a sacrificial dummy "`<<promotion`" key to eliminate empty lists
resulting from logical operations. For example:

```yaml
promo_no:   # Without promotions
  aa: alpha
  bb:
    - if:
       - true
       - tigers:
         - love cheese
    - else:
       - tigers:
         - have fleas
  cc: charlie
  dd:
    - if:
       - false
       - lose_this:
         - lost luggage
 
promo_si_1:  # With some promotions
  aa: alpha
  <<bb:
    - if:
       - true
       - tigers:
         - love cheese
    - else:
       - tigers:
         - have fleas
  cc: charlie
  <<dd:
    - if:
       - false
       - lose_this:
         - lost luggage
 
promo_si_2:   # With additional promotion
  aa: alpha
  <<bb:
    - if:
       - true
       - <<tigers:
         - love cheese
    - else:
       - <<tigers:
         - have fleas
  cc: charlie
  <<dd:
    - if:
       - false
       - <<lose_this:
         - lost luggage
```

Results after applying the logic filter:

```yaml
promo_no:
  aa: alpha
  bb:
    - tigers:
      - love cheese
  cc: charlie
  dd: []
 
promo_si_1:
  aa: alpha
  tigers:
    - love cheese
  cc: charlie
 
promo_si_2:
  aa: alpha
  tigers: love cheese
  cc: charlie
```

In one more example, we'll explore the differences between following "`else`" with a
scalar (string), single and multi-element lists, and dicts. These results would be
exactly the same following "`if`" or "`elif`", except that, unlike "`else`", "`if`" and
"`elif`" must always be followed by a list.

```yaml
---
# Original input
promo_no:
  aa:
    scalar, list, dict, dict-in-list
    after else sans promotion
  bb:
    - if: [ false, false ]
    - else: tiger0 as scalar; nothing else under bb
  cc:
    - if: [ false, false ]
    - else:
       tiger1 as scalar
    - if: [ false, false ]
    - else:
       - tiger2 in single element list
    - if: [ false, false ]
    - else:
       - tiger3a in multi element list
       - tiger3b in multi element list
    - if: [ false, false ]
    - else:
       tiger4: bare dict
    - if: [ false, false ]
    - else:
       - tiger5: dict in single element list
    - if: [ false, false ]
    - else:
       - tiger6a: first dict in multi element list
       - tiger6b: second dict in multi element list
  dd: delta
promo_si:
  aa:
    scalar, list, dict, dict-in-list
    after else with promotion
  <<bb:
    - if: [ false, false ]
    - else: tiger0 as scalar; nothing else under bb
  <<cc:
    - if: [ false, false ]
    - else:
       tiger1 as scalar
    - if: [ false, false ]
    - else:
       - tiger2 in single element list
    - if: [ false, false ]
    - else:
       - tiger3a in multi element list
       - tiger3b in multi element list
    - if: [ false, false ]
    - else:
       tiger4: bare dict
    - if: [ false, false ]
    - else:
       - tiger5: dict in single element list
    - if: [ false, false ]
    - else:
       - tiger6a: first dict in multi element list
       - tiger6b: second dict in multi element list
  dd: delta
```
Results:

```yaml	
---
# Without promption
promo_no:
   aa: scalar, list, dict, dict-in-list after else sans promotion
   bb:
     - tiger0 as scalar; nothing else under bb
   cc:
     - tiger1 as scalar
     - tiger2 in single element list
     - tiger3a in multi element list
     - tiger3b in multi element list
     - tiger4: bare dict
     - tiger5: dict in single element list
     - tiger6a: first dict in multi element list
     - tiger6b: second dict in multi element list
   dd: delta
```

```yaml 
# With promption of bb and cc
promo_si:
   aa: scalar, list, dict, dict-in-list after else with promotion
   bb: tiger0 as scalar; nothing else under bb
   cc:
     - tiger1 as scalar
     - tiger2 in single element list
     - tiger3a in multi element list
     - tiger3b in multi element list
   dd: delta
   tiger4: bare dict
   tiger5: dict in single element list
   tiger6a: first dict in multi element list
   tiger6b: second dict in multi element list
```
