# Algebraic Model Management: A Survey

Patrick Schultz¹, David I. Spivak¹, and Ryan Wisnesky²  
¹ Massachusetts Institute of Technology  
² Conexus AI

## Abstract

We survey the field of model management and describe a new model management approach based on algebraic specification.

## 1 Introduction

In this paper we survey the field of model management and describe a new model management approach based on techniques from the field of algebraic specification, with the hope of establishing an interlingua between the two fields. By “model management” we mean “meta-data intensive” database management in the sense of Bernstein & Melnik [4], which we define in Section 2. By “a new algebraic model management approach” we mean our particular way [19] [20] of specifying database schemas and instances using algebraic (equational) theories.

We first noticed a connection between model management and algebraic specification while investigating applications of category theory [3] to data integration [5]. These investigations are described in [19] and [20], and we present no substantial new results in this paper. We assume readers have basic proficiency with category theory [3], algebraic specification [17], and SQL.

**Outline.** In Section 2 we describe the traditional approach to model management and in Section 3 we describe our algebraic approach. Also in Section 3 we describe the open-source CQL (Categorical Query Language) tool, available for download at [http://categoricaldata.net](http://categoricaldata.net), which implements our approach in software. We conclude in Section 4 by comparing our approach with the traditional approach.

## 2 Model Management

To quote from Melnik [16]:

> Many challenging problems facing information systems engineering involve the manipulation of complex metadata artifacts, or models, such as database schemas, interface specifications, or object diagrams, and mappings between models. The applications that solve metadata manipulation problems are complex and hard to build. The goal of generic model management is to reduce the amount of programming needed to develop such applications by providing a database infrastructure in which a set of high-level algebraic operators, such as Match, Merge, and Compose, are applied to models and mappings as a whole rather than to their individual building blocks.

In the paragraph above the word “model” is defined to mean a metadata artifact such as a schema, which conflicts with the definition of the word “model” as a structure satisfying a theory. In this paper, we use the phrase “model management” to mean the field identified above, and use the word “model” to mean a structure satisfying a theory.

Today model management is a large sub-field of information management with a research literature containing hundreds of published articles [4]. There is a consensus in that literature [4] that model management is concerned with at least the problems described in the next sections.

### 2.1 Schema mapping

Given two database schemas S and T, the schema mapping problem [7] is to construct a “mapping” `F : S → T` that captures some user-specified relationship between S and T. Different model management systems use different notions of schema, including SQL, XML, and RDF [4]. The most common mapping formalism studied in the literature is that of “embedded dependencies” (EDs) [5]: formulae in a fragment of first-order logic with useful computational properties.

We will use SQL schemas and EDs in our examples in this section. Consider the following SQL schema S, consisting of two tables connected by a foreign key:

```sql
CREATE TABLE N2(ID INT PRIMARY KEY, age INT)
CREATE TABLE N1(ID INT PRIMARY KEY, name STRING, salary INT, f INT FOREIGN KEY REFERENCES N2(ID))
```

and the following SQL schema T, consisting of one table:

```sql
CREATE TABLE N(ID INT PRIMARY KEY, age INT, name STRING, salary INT)
```

These two SQL schemas are displayed graphically in Figure 1.

An example schema mapping `F : S → T` expressing that the target table N is the join of source tables N1 and N2 along the column f is:

∀id1,id2,a,n,s. N1(id1,n,s,id2) ∧ N2(id2,a) → N(id1,a,n,s).

Two instances satisfying the above ED are shown in Figure 1. In general, many EDs can map between two SQL schemas.

### 2.2 Query generation

Given a schema mapping `F : S → T`, the query generation problem [4] is to construct a query which converts databases on S to databases on T in a way that satisfies F. The query languages typically studied include SQL, XQuery, and various comprehension- and λ-calculi [4].

A SQL query to implement the example mapping from Section 2.1 is:

```sql
INSERT INTO N
SELECT N1.ID, N1.age, N2.name, N1.sal
FROM N1, N2
WHERE N1.f = N2.ID
```

Technically, the INSERT portion of the above SQL code is not a “query”, but rather an “update”, and in practice the code generated from a query generation task will often store the results of the query. An example of running the above SQL is shown as the left-to-right direction of Figure 1. In general, many or no SQL queries may implement a set of EDs [5]. EDs can also be directly executed by an algorithm called “the chase” [5].

### 2.3 Mapping Inversion

Given a schema mapping `F : S → T`, the mapping inversion problem [10] is to construct a schema mapping `F⁻¹ : T → S` that undoes F with respect to query generation (i.e., the queries generated from F and F⁻¹ should be inverses).

The natural candidate ED to invert the schema mapping of Section 2.1 expresses that N projects onto N1 and N2:

∀id1,a,n,s. N(id1,a,n,s) → ∃id2. N1(id,n,s,id2) ∧ N2(id2,a)

and a possible SQL implementation of this ED is:

```sql
INSERT INTO N2
SELECT ID, age
FROM N

INSERT INTO N1
SELECT ID, name, sal, ID
FROM N
```

However, the above ED is not an inverse to the ED of Section 2.1, as is seen by taking ∅ = N1 ≠ N2. Indeed, it is rare for an ED, or set of EDs, to be invertible, and weaker notions of inverse, such as “quasi-inverse” [10], are common in the literature [10]. An example of running the above SQL is shown as the right-to-left direction of Figure 1.

### 2.4 Mapping Composition

Given schema mappings `F : S → T` and `G : T → U`, the mapping composition problem [8] is to construct a schema mapping `G◦F : S → U` that is equivalent with respect to the query generation problem (i.e., running the query generated from `G◦F` should have the same effect as running the query generated from G on the results of the query generated from F).

The composition of the ED from Section 2.1 with the ED from Section 2.3 is:

∀id1,id2,n,s,a. N1(id1,n,s,id2) ∧ N2(id2,a) → ∃x. N1′(id,n,s,x) ∧ N2′(x,a)

where N1’, N2’ are target “copies” of source tables N1, N2. This composed ED is not the identity, thereby showing that the ED from Section 2.3 does not invert the ED from Section 2.1. In the case of EDs, composed mappings may not exist [8], but some restrictions and extensions of EDs are closed under composition [8].

### 2.5 Schema matching

Given two database schemas S and T, the schema matching problem [5] is to automatically find “correspondences” between S and T and to automatically infer schema mappings S → T from these correspondences. In general, inference of entire mappings cannot be fully automated and the focus of the matching problem is to reduce the human effort required to construct a schema mapping by e.g., suggesting partial mappings that can be completed by users. There are many techniques for schema matching ranging from comparison of column names by string similarity to machine learning algorithms; for an overview, see [5]. In the example from Section 2.1, two correspondences that are easy to automatically find are (N1, N) and (N2, N) and tools such as Clio [14] can create the ED from Section 2.1 from these two correspondences.

### 2.6 Further References

In this paper we will focus on the problems described in the previous sections, but many other problems are studied in the model management literature [4], and many of these problems are related to algebraic specification. For example, schema/instance merge problems [4], which arise often in data integration scenarios [2], can be formalized as pushouts in suitable categories of schemas/instances [20], and such pushouts are related to model-theoretic concepts such as model amalgamation [15].

Many software products solve model management problems [4], including ETL (Extract, Transform, Load) tools [5], which extract data from separate databases, apply user-specified transformations, and then load the result into a target system such as a data warehouse; query mediators [5], which answer queries about a “virtual” integrated database by combining queries about separate source databases; and visual schema mapping tools [14] which allow users to create schema mappings by visually connecting related schema elements with lines, as shown in Figure 2.

There have been at least two attempts to provide a “meta semantics” for model management operations. In [16] Melnik gives a “state based” meta semantics to some of the above operations by defining a schema mapping S → T to be an arbitrary binary relation between instances on S and instances on T; the ED-based semantics described above is an instantiation of this meta semantics. In [2] and [13] the authors give an “institution theoretic” meta semantics to some of the above operations by defining a schema mapping S → T to be a morphism in a suitable category of schemas; CQL’s semantics is an instantiation of this meta semantics.

Fig. 2. A schema mapping in Clio [14]

## 3 Algebraic Model Management

Our approach to model management is based on the algebraic approach to databases, data migration, and data integration we describe in [19] and [20]. Those works, and hence this work, extend a particular category-theoretic data model that originated in the late 1990s [11] and was later extended in [21] and [23] and implemented in CQL ([http://categoricaldata.net](http://categoricaldata.net)).

In the next section we describe our formalism for database schemas and instances and introduce CQL. The subsequent sections implement the model management operations from Section 2 using our formalism. In this section we abbreviate “algebraic theory” as “theory”.

### 3.1 Algebraic Databases

In our formalism [20], database schemas and instances are defined as theories of a certain kind, which we describe in the next sections. For ease of exposition, we will sometimes conflate schemas and instances as defined in our formalism with their CQL equivalents.

**Type sides**  
We first fix a theory, Ty, called the type side of our formalism. The sorts of Ty are called types and the functions of Ty are the functions that can appear in schemas and instances.

CQL allows arbitrary theories to be used as type sides. But we have found that in practice, CQL users almost always want to use the theory of an existing programming language, say java, for their type side. The ability to “bind” CQL to an existing language is particularly important in model management because input data may only be accessible through, e.g., a java API. For this reason, CQL allows a type side to be defined by specifying, for each sort s, a java class Cs and a java function `String → Cs` that tells CQL how to interpret the strings it encounters in CQL programs as objects of Cs.

An example CQL type side about integers and strings is shown in Figure 3. This type side defines a theory with two sorts and infinitely many constants – all the java strings and integers – and no equations. The java code for Int says that whenever a string x is encountered in a CQL program and a term of sort Int is required, that java’s `parseInt` function should be applied to x to yield the desired Int. The keyword `literal`, used in many places in CQL, indicates a literal (user-defined constant) definition.

**Schemas**  
A schema on type side Ty is a theory extending Ty with new sorts (called entities), new unary functions from entities to types (called attributes), new unary functions from entities to entities (called foreign keys), and new equations (called data integrity constraints) of the form ∀v : s. t = t′, where s is an entity and t, t′ are terms of the same type, each containing a single free variable v. The restrictions in the preceding sentence (e.g., no functions from types to entities) are necessary to use our formalism for model management purposes [19] [20].

Figure 4 shows the CQL schemas corresponding to Figure 1. These schemas contain no equations and are both on the type side Ty defined in Figure 3.

```cql
typeside Ty = literal {
  java_types
    String = "java.lang.String"
    Int = "java.lang.Integer"
  java_constants
    String = "return input[0]"
    Int = "return java.lang.Integer.parseInt(input[0])"
}

schema S = literal : Ty {
  entities
    N1
    N2
  foreign_keys
    f : N1 -> N2
  attributes
    name : N1 -> String
    salary : N1 -> Int
    age : N2 -> Int
}

schema T = literal : Ty {
  entities
    N
  attributes
    name : N -> String
    salary : N -> Int
    age : N -> Int
}
```

Fig. 4. CQL schemas S and T on type side Ty

**Instance I on schema S**  
An instance I on schema S is a theory extending S with new 0-ary function (constant) symbols called generators and non-quantified equations. An example CQL instance on schema S (Figure 5) is shown below.

```cql
instance I = literal : S {
  generators
    1 2 3 : N1
  equations
    name(1) = Alice
    salary(1) = 100
    age(f(1)) = 20
    name(2) = Bob
    salary(2) = 250
    age(f(2)) = 20
    name(3) = Sue
    salary(3) = 300
    age(f(3)) = 30
}
```

Fig. 5. CQL instance I on schema S

Fig. 6. Initial algebra for CQL instance I

The intended meaning of an instance I, written `I`, is the term model (i.e., initial algebra) for I which contains, for each sort s, a carrier set consisting of the closed terms of sort s modulo provability in I. A morphism of instances `I → J` is a homomorphism (natural transformation) of algebras `I → J`.

Figure 6 shows the meaning of the instance I from Figure 5 in the CQL tool. The CQL tool visually displays term models as sets of tables, one per entity e, each with an ID column corresponding to the carrier set for e. The tables in Figure 6 are isomorphic to the left tables in Figure 1.

In the following sections we implement the model management operations from Section 2 using the preceding definitions of schema and instance.

### 3.2 Schema mapping

Given schemas S, T, the schema mapping problem (Section 2.1) is to construct a “mapping” `F : S → T` that captures some relationship between S and T.

Let S and T be CQL schemas on the same type side Ty. A CQL schema mapping `F : S → T` is defined as a “derived signature morphism” [18] from S to T that is the identity on Ty. That is, `F : S → T` assigns to each entity e ∈ S an entity `F(e)` ∈ T, and to each attribute / foreign key `f : s → s′` a term `F(f)`, of type `F(s′)` and with one free variable of type `F(s)`, in a way that respects equality: if `S ⊢ t = t′`, then `T ⊢ F(t) = F(t′)`.

We have found that many mappings arising in practice cannot be expressed using plain signature morphisms and require the more general notion of “derived” signature morphism. Whereas a schema mapping in Section 2.1 was an ED (formula in a fragment of first-order logic), which induces a single binary satisfaction relation between instances, CQL schema mappings are derived signature morphisms and induce three relations between instances, which we will describe in the next section.

An example CQL schema mapping `F : S → T` is shown in Figure 7, where CQL schemas S and T are defined in Figure 4. This mapping is also shown graphically in Figure 1.

```cql
mapping F = literal : S -> T {
  entities
    N1 -> N
    N2 -> N
  foreign_keys
    f -> lambda x:N. x
  attributes
    name -> lambda x:N. name(x)
    salary -> lambda x:N. salary(x)
    age -> lambda x:N. age(x)
}
```

Fig. 7. CQL schema mapping `F : S → T`

Note that unlike Section 2.1, where there was a single query associated with a schema mapping (ED), in our algebraic approach there are three queries, one for each of `∆F`, `ΣF`, `ΠF`. The conditions under which `∆F`, `ΣF`, `ΠF` can be expressed in SQL and vice-versa are characterized in [23].

Although it is possible to give explicit formulae to define `∆F`, `ΣF`, `ΠF` [19], we instead give examples in Figures 1 and 8. Note that in these examples we are not showing instances (theories) as defined in Section 3.1; we are showing term models. For this reason, we surround `∆F`, `ΣF`, `ΠF` with denotation brackets in these examples. In addition, as adjoints `∆`, `Σ`, `Π` are only defined up to unique isomorphism, so we arbitrarily make up names for IDs and in these examples.

Figures 1 and 8 show a CQL schema mapping F which takes two distinct source entities, N1 and N2, to the target entity N. The `∆F` functor projects in the opposite direction of F: it projects columns from the single table for N to two separate tables for N1 and N2, similar to `FROM N AS N1` and `FROM N AS N2` in SQL. When there is a foreign key from N1 to N2, the `∆F` functor populates it so that N can be recovered by joining N1 and N2. The `ΠF` functor takes the cartesian product of N1 and N2 when there is no foreign key between N1 and N2, and joins N1 and N2 along the foreign key when there is. The `ΣF` functor disjointly unions N1 and N2; because N1 and N2 are not union compatible (have different columns), `ΣF` creates null values. When there is a foreign key between N1 and N2, `ΣF` merges the tuples that are related by the foreign key, resulting in a join. As these examples illustrate, `∆F` can be thought of as projection, `ΠF` can be thought of as a product followed by a filter (which can result in a join), and `ΣF` can be thought of as a disjoint union (which does not require union-compatibility) followed by a merge (which can also result in a join).

### 3.3 Query generation

Given a mapping `F : S → T`, the query generation problem (Section 2.2) is to use F to construct a query which converts databases on S to databases on T.

In our formalism, the database instances and morphisms on a schema S constitute a category, denoted `S–Inst`, and a schema mapping `F : S → T` induces a functor `ΣF : S–Inst → T–Inst` defined by substitution. The functor `ΣF` has a right adjoint, `∆F : T–Inst → S–Inst`, which corresponds to the “model reduct functor” when our formalism is described in institution-theoretic terms [2]. The functor `∆F` has a right adjoint, `ΠF : S–Inst → T–Inst`. See [19] for proof that `∆F` always has left and right adjoints. As adjoints, `∆F`, `ΠF` preserve limits and `∆F`, `ΣF` preserve colimits, implying many useful properties; for example, `ΣF (I + J) ≅ ΣF (I) + ΣF (J)` and `ΠF (I × J) ≅ ΠF (I) × ΠF (J)`.

## 4 Conclusion

When comparing our algebraic approach to model management with other approaches originating in relational database theory [1], it is important to note that our databases are “deductive databases” [1]. That is, we define databases “intentionally”, as sets of equations, rather than as sets of tables. As such, care must be taken when mediating between our definitions and relational definitions. For example, our instances can be “inconsistent” in the sense that an instance can prove `1 = 2` for two distinct constant symbols 1 and 2. Such situations are often, but not always [12], errors, and the CQL tool checks for such situations using standard techniques based on “conservative theory extensions” [12]. In addition, our schemas do not define a set of constants (a “domain”) that all the instances on that schema share, as is customary in relational database theory [7]. Hence our approach is closer in spirit to traditional logic [6] than database theory [1].

There are many connections between our algebraic approach to model management and the ED-based approach described in Section 2. EDs are more expressive than our purely equational data integrity constraints and can be added to our formalism in a simple way, described in [22] (although in [22], EDs are called “lifting problems”). In ED-based approaches the “chase” [5] operation has a similar semantics to our Σ operation, and a formal comparison between the chase and Σ is forthcoming.

