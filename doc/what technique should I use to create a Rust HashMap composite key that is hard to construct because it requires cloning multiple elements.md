I am doing some dynamic programming, and I'd like to store already-computed values in a HashMap. Unfortunately, the key is a composite value, and somewhat expensive to construct:

```rust
#[derive(Eq, PartialEq, Hash)]
struct CostKey {
    roots: Vec<usize>,
    plans: Vec<Option<RegionPlanCandidate>>,
}

//used like
pub(super) fn cost_for(
    &self,
    roots: &[usize],
    plans: &[Option<RegionPlanCandidate>],
) -> PlanCostLog {
    let key = CostKey {
        roots:Vec::from(roots),
        plans:Vec::from(plans),
    };
    if let Some(cost) = self.cost_cache.borrow().get(&key) {
        return (*cost).clone();
    }

    ...

    let rval = PlanCostLog::CrackSum(cost_log);
    self.cost_cache.borrow_mut().insert(key, rval.clone());
    rval
}
```
Even with this expensive implementation, I was able to cut the compute time for one of my examples by half. But cargo flamegraph shows that I'm still spending a non-trivial amount of time on the Vec::from calls.

If the key was not composite, and was just a reference, then the .raw_entry_mut().from_key(&key) would apply, but the nature of my key is problematic.

Theoretically, a map should be able to check Eq and Hash using just the borrowed roots and plans but I am not sure how to accomplish it with the existing APIs. How can I speed up the gets and only clone the slices when I need to insert?

Given your types, there isn't a huge amount you can do to improve the efficiency of HashMap::get here.

answer1

If your types were simpler, the way you could try to do this is to create another type which doesn't own its data, but hashes the same and can be compared for equality with CostKey, something like this:
```rust

#[derive(Eq, PartialEq, Hash)]
struct CostKeyRef<'a> {
    roots: &'a [usize],
    plans: &'a [Option<RegionPlanCandidate>],
}

impl<'a> PartialEq<CostKey> for CostKeyRef<'a> {
    fn eq(&self, other: &CostKey) -> bool {
        self.roots == &other.roots && self.plans == &other.plans
    }
}
```

However, a problem arises when you try to implement Borrow<CostKeyRef<'a>> for CostKey. This is required for various HashMap methods, but can't be implemented because the types contain two fields. There isn't a way to coerce a &CostKey into a &CostKeyRef because their layouts are just incompatible.

You may be able to alter your types so that this is possible, but this is not advisable for a Rust beginner as it would require a good understanding of how data and, in particular, references and fat pointers are laid out in memory.

So what can you do?

Well, if your hash map is relatively small, you can use a linear probe instead. Exactly how small "relatively small" is will need to be discovered through measurement, but it will certainly be larger than 100 items, and quite possibly in the 1000's or more, depending on how much overhead all of that allocation actually has.

Using the same type as above (and simplifying your code in general, for the sake of illustration), you can do something like this:
```rust

use std::collections::HashMap;

#[derive(Eq, PartialEq, Hash)]
struct CostKey {
    roots: Vec<usize>,
    plans: Vec<Option<RegionPlanCandidate>>,
}

#[derive(Eq, PartialEq, Hash, Clone)]
struct RegionPlanCandidate;

struct Thing {
    cost_cache: HashMap<CostKey, i64>,
}

impl Thing {
    fn cost_for(&mut self, roots: &[usize], plans: &[Option<RegionPlanCandidate>]) -> i64 {
        let key = CostKeyRef { roots, plans };
        if let Some(cost) =
            self.cost_cache
                .iter()
                .find_map(|(cost_key, cost)| (&key == cost_key).then(|| *cost))
        {
            return cost;
        }
        let rval = 12345;
        self.cost_cache.insert(
            CostKey {
                roots: roots.to_vec(),
                plans: plans.to_vec(),
            },
            12345,
        );
        rval
    }
}
```

answer2

As a stopgap measure, I have created a struct DualKeyHashMap that provides the features I need. I cloned many fragments from the regular HashMap implementation, and it has only the two methods I need.

```rust
use hashbrown::raw::RawTable;

use hashbrown::hash_map::DefaultHashBuilder;
use std::borrow::Borrow;
use std::hash::{BuildHasher, Hash};
use std::mem;

/// copy of hashbrown::hash_map::make_hash()
#[cfg_attr(feature = "inline-more", inline)]
pub(crate) fn make_hash<K, Q, S>(hash_builder: &S, val: &Q) -> u64
where
    K: Borrow<Q>,
    Q: Hash + ?Sized,
    S: BuildHasher,
{
    use core::hash::Hasher;
    let mut state = hash_builder.build_hasher();
    val.hash(&mut state);
    state.finish()
}

/// copy of hashbrown::hash_map::make_hasher()
#[cfg_attr(feature = "inline-more", inline)]
pub(crate) fn make_hasher<K, Q, V, S>(hash_builder: &S) -> impl Fn(&(Q, V)) -> u64 + '_
where
    K: Borrow<Q>,
    Q: Hash,
    S: BuildHasher,
{
    move |val| make_hash::<K, Q, S>(hash_builder, &val.0)
}

/// Ensures that a single closure type across uses of this which, in turn prevents multiple
/// instances of any functions like RawTable::reserve from being generated
#[cfg_attr(feature = "inline-more", inline)]
fn equivalent_key<Q, K, V>(k: &Q) -> impl Fn(&(K, V)) -> bool + '_
where
    K: Borrow<Q>,
    Q: ?Sized + Eq,
{
    move |x| k.eq(x.0.borrow())
}

//

pub trait AlternateKey<O>: Hash {
    fn eq(&self, arg: &O) -> bool;
}

//

pub struct DualKeyHashMap<K, V, S = DefaultHashBuilder> {
    hash_builder: S,
    base: RawTable<(K, V)>,
}

impl<K: Hash + Eq, V, S: BuildHasher+Default> DualKeyHashMap<K, V, S> {
    pub fn new() -> DualKeyHashMap<K, V, S>
    {
        Self::default()
    }
}

impl<K: Hash + Eq, V, S: BuildHasher+Default> Default for DualKeyHashMap<K, V, S> {
    fn default() -> DualKeyHashMap<K, V, S>
    {
        DualKeyHashMap {
            hash_builder: Default::default(),
            base: Default::default(),
        }
    }
}
impl<K: Hash + Eq, V, S: BuildHasher> DualKeyHashMap<K, V, S> {

    pub fn insert(&mut self, key1: K, val1: V) -> Option<V> {
        let hash = make_hash::<K, _, S>(&self.hash_builder, &key1);
        //println!("{}", hash);

        if let Some((_, item)) = self.base.get_mut(hash, equivalent_key(&key1)) {
            Some(mem::replace(item, val1))
        } else {
            self.base.insert(
                hash,
                (key1, val1),
                make_hasher::<K, _, V, S>(&self.hash_builder),
            );
            None
        }
    }

    #[inline]
    pub fn get<Q: ?Sized>(&self, k: &Q) -> Option<&V>
    where
        K: Borrow<Q>,
        Q: Hash + Eq,
    {
        // Avoid `Option::map` because it bloats LLVM IR.
        match self.get_inner(k) {
            Some(&(_, ref v)) => Some(v),
            None => None,
        }
    }

    fn get_inner<Q: ?Sized>(&self, k: &Q) -> Option<&(K, V)>
    where
        K: Borrow<Q>,
        Q: Hash + Eq,
    {
        let hash = make_hash::<K, Q, S>(&self.hash_builder, k);
        self.base.get(hash, equivalent_key(k))
    }

    pub fn get2<M>(&self, k: &M) -> Option<&V>
    where
        M: AlternateKey<K>,
    {
        let hash = make_hash::<M, M, S>(&self.hash_builder, k);
        match self
            .base
            .get(hash, |(k2, _)| <M as AlternateKey<K>>::eq(k, &k2))
        {
            Some(&(_, ref v)) => Some(v),
            None => None,
        }
    }

    pub fn len(&self)->usize
    {
        self.base.len()
    }
}
```

I do not consider this a proper answer since it requires an entirely new hash map implementation which is extremely incomplete.

