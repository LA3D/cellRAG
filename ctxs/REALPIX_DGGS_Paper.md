THE RHEALPIX DISCRETE GLOBAL GRID SYSTEM
ROBERT GIBB, ALEXANDER RAICHEV, AND MICHAEL SPETH
Abstract. In this paper we extend the HEALPix discrete global grid system (DGGS)
to ellipsoids of revolution, thereby broadening its possible applications. Elaborating
on the work of Calabretta and Roukema [Mapping on the HEALPix grid, Monthly
Notices of the Royal Astronomical Society 381 (2007), no. 2, 865872.], we also rearrange
the HEALPix map projection and build a new DGGS on top of it, which we call the
rHEALPix DGGS. The rHEALPix DGGS has all the key features of the HEALPix
DGGS, and its planar projection consists of horizontal-vertical aligned nested square
grids, which makes it easy to understand and display. We present all the formulas and
algorithms necessary for a basic implementation of the rHEALPix DGGS and link to
our own open source implementation thereof.
1. Introduction
A discrete global grid is a finite partition of the surface of an ellipsoid (which could
be a sphere) along with a set of distinguished points, one point in each partition element.
A partition element is called a cell and its unique associated point is called a nucleus
herein
∗
. A discrete global grid system (DGGS) is a sequence of discrete global
grids, usually of increasingly finer resolution. DGGSs are used to store and analyze ellip-
soidal spatial data to study celestial and planetary phenomena such as cosmic microwave
background radiation and global warming [SWK03].
The HEALPix DGGS was introduced in [GHB
+
05] to study cosmic microwave back-
ground radiation and has the following key features.
(i) It is hierarchical, congruent, aligned for odd values of an integer parameter N
side
, and
constant aperture, which makes it easy to implement with efficient data structures.
(ii) At every resolution its grid cells have equal areas, which provides equal probabilities
for statistical analyses.
(iii) At each resolution its k nuclei lie on only Θ(
√
k) parallels of latitude, which makes
computing spherical harmonics fast.
(iv) Its planar projection has low average angular and linear distortion, which means
that on average it distorts angles and lengths of small regions minimally.
In this paper we extend the HEALPix DGGS, which was initially defined for spheres
only, so that:
(v) It can be used on ellipsoids of revolution such as the WGS84 ellipsoid, which is also
used by the Global Positioning System and most satellite-based sensors.
We do this while preserving features (i)–(iv) and thereby broaden the possible applications
of the HEALPix DGGS.
Date: January 26, 2013.
Key words and phrases. HEALPix, rHEALPix, discrete global grid system.
Thanks to Kevin Sahr and Martin Lambers for reading an early draft of this paper and for offering
constructive comments that have improved its quality.
∗ 
We prefer the term ‘nucleus’ over the more common term ‘cell center’, because the latter can be
misleading: a cell’s nucleus is often not its centroid.
1

Elaborating on the work of [CR07, Section 3.1], we also rearrange the underlying
HEALPix map projection into what we call the rHEALPix map projection and detail a
DGGS for ellipsoids of revolution based upon it, which we call the rHEALPix DGGS.
The rHEALPix DGGS also has features (i)–(v) and, in addition, satisfies:
(vi) Its planar projection consists of horizontal-vertical aligned nested square grids,
which makes it easy to understand and display.
So by the criteria 1, 2, 6, 8, 9, 12, and 14 of [KSWS99], the rHEALPix DGGS is a good
choice of DGGS for storage and analysis of ellipsoidal spatial data and an especially good
choice when harmonic analysis is required.
The rHEALPix DGGS can be construed as a mapping of an ellipsoid of revolution onto
a regular polyhedron, namely a cube, followed by a symmetric hierarchical partitioning
of the polyhedral faces along with a choice of nuclei, followed by the inverse mapping of
the result back onto the ellipsoid. Thus the rHEALPix DGGS is an example of a geodesic
DGGS, specifically a cubic geodesic DGGS [SWK03].
To the best of our knowledge, the rHEALPix DGGS and other variants of the HEALPix
DGGS are the only DGGSs satisfying properties (i)–(vi)
†
. Other prominent DGGSs that
come close are the COBE DGGS [CO75], its variant [OL76], and the SAND DGGS [AS00],
all of which are cubic geodesic DGGSs. They satisfy properties (i) (they can be made
aligned by introducing an N
side 
parameter), (ii) (approximately so for COBE), (iv), (v)
(after adjustment), and (vi), but fail (iii) because they have Θ(k) nucleus parallels. Check
SAND claim. Similarly, Snyder’s equal-area cubic projection [Sny92] can be rotated and
partitioned to yield a cubic DGGS satisfying all properties (i)–(vi) except (iii), because
it also will have Θ(k) nucleus parallels.
Of course, in applications where features (i)–(vi) are not required, the rHEALPix
DGGS might not be appropriate. For example, fluid flow models typically require grids
with uniform adjacency, ones where all cells that share boundaries share edge boundaries.
In that case, grids with hexagonal rather than quadrilateral cells are appropriate [SWK03,
page 127].
Our implementations of the extended HEALPix and rHEALPix map projections can be
found in the forthcoming release (after 4.8) of the PROJ.4 cartographic library, and our
Python implementation of the rHEALPix DGGS can be found at Landcare Research’s
git repository http://code.scenzgrid.org/index.php/p/scenzgrid-py/. Both of these im-
plementations are open source and based upon the formulas and algorithms included
herein.
2. Defining the rHEALPix DGGS
Choose a base ellipsoid of revolution, hereafter referred to as the ellipsoid, and place
a geodetic longitude-latitude coordinate frame upon it. For instance, the ellipsoid could
be the WGS84 ellipsoid with the WGS84 coordinate frame. Let R
q 
denote the radius
of the ellipsoid’s authalic sphere, the unique sphere having the same surface area as the
ellipsoid. In case of the WGS84 ellipsoid, R
q 
= 6, 374, 581.4671 m. For mathematical
convenience, we measure all angles in radians and all lengths in meters unless indicated
otherwise.
Extending the HEALPix DGGS from a sphere to the ellipsoid while preserving features
(i)–(iv) is straightforward. Simply (a) map the ellipsoid onto its authalic sphere in the
† 
As also mentioned in [CR07, Section 3], the HEALPix projection image can be rotated by 45
◦ 
and
then partitioned with horizontal-vertical aligned nested square grids to produce another cubic DGGS
with features (i)–(vi). The drawback there, though, is that all parallels of latitude project to the plane
at 45
◦ 
angles, which can confuse human viewers. So we do not pursue that DGGS here.
2

standard area-preserving fashion, (b) map the authalic sphere onto the plane with the
HEALPix projection, (c) define the usual HEALPix planar grid system, and (d) map this
grid system back to the ellipsoid by applying inverses. This process preserves features
(i)–(iv) of the HEALPix DGGS, because the map in step (a) preserves local areas, sends
meridians to meridians, parallels to parallels, and has low average distortion. For the
mathematical details, see Appendix A.
To define the rHEALPix DGGS, we also follow process (a)–(d) but instead use a
rearranged version of the HEALPix projection in (b) and a simpler horizontal-vertical
aligned square grid system in (c). More specifically, for step (b) we map the authalic
sphere to the plane via the (n, s)-rHEALPix projection for some choice of integers 0 ≤
n, s ≤ 3; see Figure 1(b). For the mathematical details see Appendix B.
(a)
(b)
Figure 1. Projections of the WGS84 ellipsoid with a π/16 map graticule,
axes numbered in units of authalic radius, and coastlines drawn. (a) The
HEALPix map projection with polar triangles shaded and numbered. (b)
The (1,2)-rHEALPix map projection.
3

Just like the HEALPix projection, the (n, s)-rHEALPix projection is equiareal —by
which we mean equal areas on the sphere map to equal areas on the plane— with area
scaling factor 3π/8.
For step (c), recursively construct a sequence G
0
, G
1
, G
2
, . . . of nested grids on the
planar projection as follows. Let G
0 
be the set of 6 squares of width R
q
π/2 whose
upper left vertices lie at R
q
(−π + nπ/2, 3π/4), R
q
(−π, π/4), R
q
(−π/2, π/4), R
q
(0, π/4),
R
q
(π/2, π/4), and R
q
(−π +sπ/2, −π/4). Choose an integer N
side 
≥ 2 and for each integer
i ≥ 0 let G
i+1 
be the grid obtained by refining each square of grid G
i 
into N
side 
× N
side
isomorphic sub-squares. Then each grid G
i 
comprises 6 · N 
2i
side 
isomorphic square cells,
each having side length R
q
(π/2)N 
−i
side 
and area R
2
q 
(π
2
/4)N 
−2i
side 
. Define the nucleus of each
cell to be its centroid. Call each planar grid G
i 
the (n, s)-rHEALPix planar grid
of resolution i, and call the nested sequence G := (G
0
, G
1
, G
2
, . . .) of planar grids
the (n, s)-rHEALPix planar grid hierarchy; see the left side of Figure 2. Strictly
speaking, G is not a DGGS, since each of its grids does not partition the ellipsoid but
rather a planar projection of it.
Map each planar grid G
i 
and its nuclei set back to the ellipsoid via the inverse (n, s)-
rHEALPix projection, call the resulting ellipsoidal grid G
′
i 
the (n, s)-rHEALPix ellip-
soidal grid of resolution i, and call the nested sequence G
′ 
:= (G
′
0
, G
′
1
, G
′
2
, . . .) of
ellipsoidal grids the (n, s)-rHEALPix ellipsoidal grid hierarchy; see the right side of
Figure 2. This latter hierarchy is the (n, s)-rHEALPix DGGS. Each grid G
′
i 
comprises
6 · N 
2i
side 
cells of four different shapes types detailed in Section 4. All resolution i cells have
equal area, namely R
2
q 
(π
2
/4)N 
−2i
side 
/(3π/8) = R
2
q 
(2π/3)N 
−2i
side 
, since the (n, s)-rHEALPix
map projection is equiareal with area scaling factor 3π/8. Note that the nucleus of an
ellipsoidal cell, being the inverse image of the nucleus of its corresponding planar cell,
is in general not its ellipsoidal centroid; see Section 7 for a calculation of the centroid.
Note also that for the area R
2
q 
(2π/3)N 
−2i
side 
of an ellipsoidal cell to be at most A square
meters, the resolution resolution i must be at least dlog(R
2
q 
(2π/3)A
−1
)/(2 log N
side
)e. For
example, for N
side 
= 3 the cells of G
′
i 
have area no greater than one square meter when
i ≥ 15; G
′
15 
has 6 · 9
15 
= 1, 235, 346, 792, 567, 894 cells.
Besides being hierarchical and recursive, the hierarchies G and G
′ 
are both congruent,
that is, each resolution i cell is a union of resolution i + 1 cells; when N
side 
is odd the
hierarchies G and G
′ 
are both aligned, that is, each resolution i cell nucleus is the nucleus
of a resolution i + 1 cell; the hierarchies G and G
′ 
both have a constant aperture of
N 
2
side
, that is, each resolution i cell has N 
2
side 
times the area of its resolution i + 1 subcells.
As a running example throughout this paper we use the (1,2)-rHEALPix DGGS with
N
side 
= 3. The choice of N
side 
= 3 produces aligned hierarchies with the greatest number
of resolutions per fixed maximum areal resolution.
3. Cell ID
For reference we assign unique names to the cells of the planar hierarchy G that are
shared with the corresponding cells of the ellipsoidal hierarchy G
′
. These names echo the
tree structure of the hierarchies and are defined as follows.
Let Γ be the set of all strings beginning with one of the letters N, O, . . . , S and followed
by zero or more of the integers 0, 1, . . . , N 
2
side 
−1. In other words, Γ is the regular language
(N |O| · · · |S)(0|1| · · · |(N 
2
side 
− 1))
∗
. The identifier (ID) of a cell c from G, denoted ID(c),
is the string in Γ defined by the following recursive procedure. Assign the cells of G
0 
the
IDs N, O, P, Q, R, S from top to bottom and left to right. In particular, cell N contains
the projection of the north pole and cell S contains the projection of the south pole.
4

G
0 
G
′
0
G
1 
G
′
1
G
2 
G
′
2
Figure 2. The first three planar and ellipsoidal grids for the (1, 2)-
rHEALPix map projection with N
side 
= 3, cells color-coded, planar axes
labeled in units of authalic Earth radius, and cells labeled with their IDs.
For each resolution i cell with ID s assign its N 
2
side 
resolution i + 1 subcells the IDs
s0, s1, . . . , s(N 
2
side 
− 1) from top to bottom and left to right; see Figures 2 and 3.
5

Figure 3. A resolution i cell with ID s in the rHEALPix planar hierarchy
with N
side 
= 3. Its N 
2
side 
= 9 resolution i + 1 subcells are color-coded and
labeled in black and their row and column numbers are highlighted in red.
Henceforth we will refer to cells by their IDs and sometimes blur the distinction between
numbers and strings when the context is clear.
4. Cell Shape and Distribution
Consider the grids G
i 
and G
′
i
.
The planar grid G
i 
comprises 6 · N 
2i
side 
axis-aligned isomorphic square cells, each of side
length R
q
(π/2)N 
−i
side
. In the equatorial region of the plane, the region |y| ≤ R
q
π/4,
there are 4 · N 
2i
side 
axis-aligned isomorphic square cells arranged along N 
i
side 
rows and
4 · N 
i
side 
columns. In the polar region of the plane, the non-equatorial region, there are
2 · N 
2i
side 
square cells. Half of these lie in the north polar region (y > R
q
π/4) arranged
along N 
i
side 
rows and N 
i
side 
columns, and the other half of these lie in the south polar
region (y < −R
q
π/4) arranged along N 
i
side 
rows and N 
i
side 
columns. The columns in the
north and south coincide when n = s in the (n, s)-rHEALPix projection used.
The ellipsoidal grid G
′
i 
is symmetric with respect to a π/2 longitudinal rotation or a
π/2 latitudinal rotation of the ellipsoid and comprises 6 · N 
2i
side 
cells of equal areas and
of four different shape types. Each cell, except for the two cells centered at the poles,
has its northern edge/vertex lying on one parallel and its southern edge/vertex lying on
another parallel.
In the equatorial region of the ellipsoid, the region of authalic latitudes of magnitude at
most 41.8
◦ 
approximately, there are 4 · N 
2i
side 
cells, all of which are ellipsoidal quadrangles
with longitude-latitude-aligned edges. We call an ellipsoidal cell of this shape a quad cell;
see cell P 1 in Figure 2 for instance. Quads have IDs of the form (O|P |Q|R)(0|1| · · · |N 
2
side
−
1)
∗
, in terms of regular expressions. The nuclei of quad cells are spread unequally in
latitude across N 
i
side 
parallels, and on each such parallel there are 4 · N 
i
side 
cell nuclei
spread equally in longitude.
In the polar region of the ellipsoid, the non-equatorial region, there are 2 · N 
2i
side 
cells
of three different shape types, or two different shape types if N
side 
is even. Half of these
cells lie in the north polar region and the other lie in the south polar region. Since these
two polar grid regions are isomorphic, let us just consider the north polar region, say.
In that region, if N
side 
is odd, then there is one cell whose nucleus lies at the north pole
and whose shape is an ellipsoidal cap, that is, the boundary of the cell lies along a single
parallel. We call an ellipsoidal cell of this shape a cap cell; see cell N 4 in Figure 2 for
instance. Caps have IDs of the form (N |S)((N 
2
side 
− 1)/2)
∗
. If N
side 
is even, then there
are no cap cells. 
6

The remaining N 
2i
side 
− 1 north polar cells, or N 
2i
side 
if N
side 
is even, come in two different
shapes. A cell whose nucleus lies off the poles and at longitude −π, −π/2, 0, or π/2 (the
longitudes of the diagonal and antidiagonal of planar cells N and S) has three edges, one
of which lies on a parallel and the other two of which have equal lengths and converge
polewards to a point. We call an ellipsoidal cell of this shape a dart cell; see cell
N 6 in Figure 2 for instance. Darts have IDs of the form (N |S)((0|(N
side 
+ 1)|2(N
side 
+
1)| · · · |(N
side 
− 1)(N
side 
+ 1))
+ 
| ((N
side 
− 1)|2(N
side 
− 1)| · · · |N
side
(N
side 
− 1))
+
), excluding
cap cell IDs. All other polar cells are ellipsoidal quadrangles having their northern and
southern edges lying on two parallels. We call an ellipsoidal cell of this shape a skew quad
cell; see cell N 7 in Figure 2 for instance. The nuclei of the northern dart cells and skew
quad cells are spread unequally in latitude across bN 
i
side
/2c parallels with 4(N 
i
side 
− 2j + 1)
cell nuclei spread equally in longitude along each such parallel band j ∈ {1, . . . , bN 
i
side
/2c},
where the band numbers increase polewards. Of these 4(N 
i
side 
− 2j + 1) cells, 4 are dart
cells and the remaining 4(N 
i
side 
− 2j) are skew quad cells. So in the north polar in total
there are 4bN 
i
side
/2c dart cells and (N 
i
side 
− 1)
2 
skew quad cells, or (N 
i
side 
− 1)
2 
− 1 skew
quad cells if N
side 
is even.
Notice that the nuclei of the k := 6 · N 
2i
side 
ellipsoidal cells lie along only 2N 
i
side 
− 1 (non-
pole) parallels if N
side 
is odd and 2N 
i
side 
parallels if N
side 
is even, that is, Θ(
√
k) parallels.
This square root dependence also holds for the HEALPix DGGS. Indeed, it was designed
into it to so that computing spherical harmonics is relatively fast. Computing spherical
or ellipsoidal harmonics on the ellipsoid involves computing Legendre polynomials at each
nucleus parallel, which is recursive and slow, and so the fewer the parallels, the faster the
computation [GHB
+
05, Section 5] [Nes11]. For comparison, the COBE DGGS [CO75]
used by NASA before HEALPix has Θ(k) nucleus parallels at each grid resolution.
5. Cell Boundaries and Neighbors
Because the image of an rHEALPix map projection does not completely contain its
boundary (see Figure 1), the cells of G and G
′ 
cannot contain all their edges. Let us
declare somewhat arbitrarily that cells N and S contain none of their edges and that
cells O–R contain their left, top, and bottom edges. Recursing, let us declare that each
child cell contains its left and top edges unless it shares a boundary with its parent, in
which case it inherits it parent’s boundary. For example, cell N 5 contains its left and top
edges, and cell N 0 contains none of its edges. The figures herein depicting cells ignore
these subtleties of boundary by representing all boundaries with solid lines.
Every ellipsoidal cell c shares an edge with four other cells, which we call the neighbors
of c. Additionally, c shares a boundary point with three or four other cells, depending
if c is a dart or not, respectively. For simplicity, we do not consider these three or four
other adjacent cells neighbors of c. For a cell on the boundary between the equatorial and
polar region of the ellipsoid, the names of its neighboring cells depend on the parameters
n and s of the chosen (n, s)-rHEALPix projection, that is, on how the polar triangles of
the HEALPix map projection are combined. For example, in the (1, 2)-rHEALPix DGGS
with N
side 
= 3, the neighbors of P 0 are N 6, O2, P 3, and P 1. In the (0, 2)-rHEALPix
DGGS with N
side 
= 3, the neighbors of P 0 are N 8, O2, P 3, and P 1.
The cell adjacency structure of the ellipsoidal hierarchy G
′ 
induces a cell adjacency
structure on the planar hierarchy G that can be visualized by folding G
0 
into a cube
along its cell edges, and this is how the rHEALPix DGGS can be construed as a geodesic
DGGS; see Figure 4. 
7

Figure 4. The cell adjacency structure of G
1 
for the (1,2)-rHEALPix map
projection with N
side 
= 3.
6. Cell Vertices and Nucleus
From a planar cell’s ID we can compute its four vertices and nucleus as follows. For a
string s of length k, let s[i] denote the character of s at index i, where 0 ≤ i < k, and let
s[i :] denote the string s[i]s[i + 1] · · · s[k − 1]. Given a digit in {0, 1, . . . , N 
2
side 
− 1}, let the
row ID and column ID of the digit be its row and column index in the N
side 
× N
side
subcell matrix of Figure 3. For example, for N
side 
= 3 we have rowID(5) = 1 and
colID(5) = 2. Notice that the row and column IDs of a digit can be obtained from the
base N
side 
expansion of the digit: 5 = (12)
3
. Given a letter in {N, O, . . . , S}, let its row
and column ID be the letter itself. Let the row ID of a cell be the concatenation of the row
IDs of its individual ID characters, and let the column ID of a cell be defined analogously.
For example, for N
side 
= 3 we have rowID(Q517) = Q102 and colID(Q517) = Q211.
The last digit of a cell’s ID determines the cell’s upper left vertex relative to its parent
cell’s upper left vertex
‡
. For example, for N
side 
= 3 consider the planar cell s5 of width
w. The last digit of its ID is 5 = (12)
3
, and the cell lies 1 · w down and 2 · w right from
the upper left vertex of its parent cell s. In general, for a cell sa of resolution at least 1
and with last digit a, we have
ul(sa) = ul(s) + w(colID(a), − rowID(a)),
where w is the width of cell sa and ul(t) denotes the coordinates of the upper left vertex
of cell t. Recursing on this formula and recalling that a cell of resolution i has width
(R
q
π/2)N 
−i
side
, we get that the upper left vertex of a resolution r planar cell with ID s is
ul(s) = ul(s[0]) + (R
q
π/2)
r
∑
i=1
N 
−i
side
(colID(s[i]), − rowID(s[i])),
Alternatively we could write the x and y coordinates of ul(s), respectively, as
ul(s)
x 
= ul(colID(s)[0])
x 
+ (R
q
π/2)(0. colID(s)[1 :])
N
side
ul(s)
y 
= ul(rowID(s)[0])
y 
− (R
q
π/2)(0. rowID(s)[1 :])
N
side
For example, for N
side 
= 3 the upper left vertex of cell Q517 is
ul(P 517)
x 
= ul(P )
x 
+ (R
q
π/2)(0.211)
3 
= R
q
[−π/2 + (π/2)(2/3 + 1/9 + 1/27)]
ul(P 517)
y 
= ul(P )
y 
− (R
q
π/2)(0.102)
3 
= R
q
[π/4 − (π/2)(1/3 + 2/27)].
‡ 
Notice that a cell might not contain its upper left vertex since it lies on the cell’s boundary. For
example, cell N does not contain its upper left vertex.
8

From the upper left vertex coordinates (x, y) of a planar cell and from its width w, we
can then compute the coordinates of its remaining vertices and nucleus.
upper left vertex : (x, y)
upper right vertex : (x + w, y)
lower left vertex : (x, y − w)
lower right vertex : (x + w, y − w)
nucleus : (x + w/2, y − w/2).
Projecting these vertices onto the ellipsoid produces the vertices of the corresponding
ellipsoidal cell, if it is a quad cell or a skew quad cell. If the ellipsoidal cell is a dart or
a cap cell, one or four of the projected points, respectively, will be non-vertex boundary
points. Projecting the nucleus of a planar cell onto the ellipsoid produces the nucleus of
the corresponding ellipsoidal cell by definition.
7. Cell Centroid
By definition, the centroid of a planar cell is its nucleus and projecting this nucleus
back onto the ellipsoid yields the nucleus of the corresponding ellipsoidal cell. But in
general, the centroid of an ellipsoidal cell is not its nucleus. In other words, cell nuclei
are preserved under the rHEALPix projection but centroids are not. We can compute
the centroid of an ellipsoidal cell by integration and symmetry as follows.
Let r denote an rHEALPix map projection of the ellipsoid and observe that the average
value of a real function f (λ, φ) on an ellipsoidal cell c is
1
area of c
∫ ∫
c
f dS = 
1
area of c
∫ ∫
r[c]
(f ◦ r
−1
)(x, y) 
8
3π 
dxdy
= 
1
area of r[c]
∫ ∫
r[c]
(f ◦ r
−1
)(x, y)dxdy
when parametrized by planar coordinates (x, y). The factor 8/(3π) is the area scaling
factor of r
−1
, which is the reciprocal of the area scaling factor of r, and r[c] is the planar
projection of c. Applying this average value formula to the longitude f (λ, φ) = λ and
and latitude f (λ, φ) = φ functions gives the coordinates (
λ, φ) of the centroid of cell c:
λ = w
−2
∫ 
y
2
y
1
∫ 
x
2
x
1
r
−1
λ 
(x, y)dxdy
φ = w
−2
∫ 
y
2
y
1
∫ 
x
2
x
1
r
−1
φ 
(x, y)dxdy,
where x
1 
< x
2 
and y
1 
< y
2 
are the respective x and y extremes of the planar cell r[c],
w = x
2 
− x
1 
= y
2 
− y
1 
is its width, and r
−1
λ 
and r
−1
φ 
are the longitude and latitude
components of r
−1
, respectively.
For several ellipsoidal cell shapes we can skirt these integrals. By symmetry the centroid
of every cap cell is its nucleus, namely (−π, ±π/2). Since every quad cell is longitude-
latitude aligned, its centroid is the average of its two longitudinal extremes and the
average of its two latitudinal extremes. In particular, the longitude of its centroid is
the longitude of its nucleus. Also, every dart cell is bisected by a meridian, and so the
longitude of its centroid is the longitude of its nucleus. The following table summarizes
these observations with (λ
n
, φ
n
) denoting the cell’s nucleus.
9

ellipsoidal cell λ φ
cap λ
n 
φ
n
quad λ
n 
(φ(x
1
, y
1
) + φ(x
1
, y
2
))/2
dart λ
n 
w
−2 
∫ 
y
2
y
1
∫ 
x
2
x
1 
r
−1
φ 
dxdy
skew quad w
−2 
∫ 
y
2
y
1
∫ 
x
2
x
1 
r
−1
λ 
dxdy w
−2 
∫ 
y
2
y
1
∫ 
x
2
x
1 
r
−1
φ 
dxdy
Note that from a cell’s ID we can compute x
1
, x
2
, y
1
, y
2
, w, λ
n
, and φ
n 
by the method
in Section 6.
The integral for λ for a skew quad cell can be computed exactly. This is most easily
done by considering the HEALPix projection of the cell and using the formula for the
longitude component of the inverse HEALPix projection given in Appendix A:
λ = w
−2
∫ 
y
2
y
1
∫ 
x
2
x
1
(
x
c 
+ 
x/R
q 
− x
c
2 − 4y/(πR
q
)
)
dxdy
= w
−2
[[
I(x, y)
]
x
2
x
1
]
y
2
y
1
= w
−2
(I(x
2
, y
2
) − I(x
1
, y
2
) − I(x
2
, y
1
) + I(x
1
, y
1
))
I(x, y) = 
π
8 
x(2R
q
x
c 
− x) log
(
1 − 
2y
πR
q
)
+ x
c
xy
x
1 
= x
0 
− w/2
x
2 
= x
0 
+ w/2
y
1 
= |y
0
| − w/2
y
2 
= |y
0
| + w/2
x
c 
= −
3π
4 
+ 
π
2
⌊
2(x
0
/R
q 
+ π)
π
⌋
(x
0
, y
0
) = the HEALPix projection of the cell’s nucleus.
The integral for φ for a skew quad cell or a dart cell, however, must be computed
approximately/numerically, because there is no closed form formula for r
−1
φ 
(x, y), because
there is no closed form formula for latitude in terms of authalic latitude; see Appendix A.3.
8. Cell from Point
Given a point (x, y) in the rHEALPix planar image and an integer i ≥ 0, we can
compute the ID of the resolution i cell that contains (x, y) as follows. First compute by
cases the resolution 0 cell c containing (x, y). Then compute the horizontal and vertical
distances from (x, y) to ul(c):
(∆x, ∆y) := (| ul(c)
x 
− x|, | ul(c)
y 
− y|).
Then find out how far along the width R
q
π/2 of c lie ∆x and ∆y by computing the
base N
side 
expansions (0.s)
N
side 
and (0.t)
N
side 
of ∆x/(Rπ/2) and ∆y/(Rπ/2), respectively.
Truncating s and t at i digits then gives the column and row IDs, respectively, of the
resolution i cell d containing (x, y):
ID(d) = ID(c)(N
side
t[0] + s[0])(N
side
t[1] + s[1]) · · · (N
side
t[l] + s[l]).
10

We can do the same for points on the ellipsoid by projecting them into the plane and
then carrying out the computation above.
9. Cell from Region
Given a region A in an rHEALPix planar image that is bounded by an axis-aligned
rectangle specified by, say, upper left and lower right vertices u and v, respectively, we can
compute the smallest cell c wholly containing A, if it exists, as follows. For i from 0 to ∞,
compute the IDs of the resolution i cell containing u and the resolution i cell containing
v, and halt at the first index r at which the two IDs disagree. Then the longest (length
r) common prefix of these two IDs will be the ID of c. In case the longest common prefix
is the empty string, then no cell entirely contains the region A.
We can do the same analysis for regions on the ellipsoid bounded by an longitude-
latitude aligned ellipsoidal quadrangle by first projecting the quadrangle into the plane,
calculating its axis-aligned bounding rectangle, and then proceeding as above.
10. Conclusion
Having now detailed enough of the rHEALPix DGGS for a basic implementation, let
us stop here. As we have seen through properties (i)–(vi) enumerated in Section 1, the
rHEALPix DGGS is a good choice of DGGS for the storage and analysis of spatial data
on an ellipsoid of revolution, especially when harmonic analysis is required.
No single DGGS is ideal for all tasks, however [KSWS99], and the rHEALPix DGGS
is no exception. As we mentioned in the introduction, it is not ideal in applications that
require grids with uniform adjacency. We also suspect that it in not ideal in applications
that require near equidistance of cell nuclei or extreme cell compactness. In that case,
grids with triangular or hexagonal cells might be better [GKWS08]. In future work we
would like to explore this suspicion by computing nuclei spacing and compactness metrics
for the rHEALPix DGGS and comparing them to other DGGSs.
Appendix A. The HEAPix Projection
The HEALPix map projection of a given sphere is the H = 4, K = 3 member of an
infinite family (parametrized by positive integers H and K) of map projections of the
sphere. As observed in [CR07], it is the hybrid of a rescaled Lambert cylindrical equal
area projection on equatorial latitudes |φ| ≤ φ
0 
:= arcsin(2/3) and a 4-times interrupted
Collignon projection on polar latitudes |φ| > φ
0
; see Figure 1(a).
Some of its features are:
(i) It is an equiareal projection (by which we mean equal areas on the sphere map to
equal areas on the plane) with area scaling factor 3π/8 and low average angular and
linear distortion.
(ii) Parallels of latitude project to horizontal lines.
(iii) A parallel of latitude uniformly divided projects to a horizontal line uniformly di-
vided.
(iv) Meridians of longitude project to vertical lines in the equatorial region and lines
converging polewards in the polar region.
A.1. Formulas for the Sphere. Let S
2
R 
⊂ R
3 
denote the sphere of radius R centered at
the origin. By a map projection of the sphere, we mean an injective function from S
2
R 
to
the plane R
2
. In cartography, points on the sphere are usually described in terms of their
11

longitude and geodetic latitude, that is, via the bijective parametrization p : D → S
2
R
defined by
p(λ, φ) = R(cos λ cos φ, sin λ cos φ, sin φ),
where D = [−π, π) × (−π/2, π/2) ∪ {(−π, π/2), (−π, π/2)}. Specifying an injective func-
tion f : D → R
2
, which we call a signature function, then uniquely determines a map
projection f ◦ p
−1 
of the sphere.
The signature function of the HEALPix map projection is h
R 
:= 
¯
R ◦ h
1
, where
h
1
(λ, φ) =
{
(λ, 
3π
8 
sin φ) if |φ| ≤ φ
0
(λ
c 
+ (λ − λ
c
)σ, sgn(φ)
π
4 
(2 − σ)) else
σ = 
√
3(1 − | sin φ|)
λ
c 
= −
3π
4 
+ 
π
2
⌊
2(λ + π)
π
⌋
¯
R(x, y) = (Rx, Ry).
See Figure 1(a). Consequently, its inverse is h
−1
R 
= h
−1
1 
◦ 
¯
R
−1
, where
h
−1
1 
(x, y) =





(
x, arcsin 
( 
8y
3π
)) 
if |y| ≤ 
π
4
(x
c 
+ 
x−x
c
τ 
, sgn(y) arcsin(1 − 
τ 
2
3 
)) if 
π
4 
< |y| < 
π
2
(−π, 
π
2 
) else,
τ = 2 − 
4|y|
π
x
c 
= −
3π
4 
+ 
π
2
⌊
2(x + π)
π
⌋
¯
R
−1
(x, y) =
( 
x
R 
, 
y
R
)
.
These formulas come from [CR07].
A.2. Distortion of the Sphere. Every map projection necessarily distorts angles, lengths,
or areas of the sphere [Eul78]. These distortions can be quantified by a map projection’s
local scales, which we now compute for the HEALPix map projection according to [BS95,
Section 1.6] and [Sny87, Section 4].
For this we need the partial derivatives of p and h := h
R 
on the interior of D:
∂p
∂λ 
= R(− sin λ cos φ, cos λ cos φ, 0)
∂p
∂φ 
= R(− cos λ sin φ, − sin λ sin φ, cos φ)
∂h
∂λ 
=
{
R(1, 0) if |φ| ≤ φ
0
R(σ, 0) else
∂h
∂φ 
=
{
R(0, 
3π
8 
cos φ) if |φ| ≤ φ
0
R(− sgn(φ)(λ − λ
c
)( 
3
2σ 
) cos φ, 
3π
8σ 
cos φ) else.
Notice that ∂p/∂λ and ∂p/∂φ are orthogonal since their dot product is 1. Notice also
that cos φ/σ tends to 
√
2/3 as |φ| approaches π/2.
12

The local linear scale of a map projection at a given point and in a given direction
is the reciprocal of the ratio of the length of an infinitesimal segment on the sphere to
the length of the projected segment in the plane. The HEALPix local linear scales along
parallels of latitude and meridians of longitude, respectively, are
s
P 
=
∣
∣
∣
∣
∂h
∂λ
∣
∣
∣
∣
∣
∣
∣
∣
∂p
∂λ
∣
∣
∣
∣
−1
=
∣
∣
∣
∣
∂h
∂λ
∣
∣
∣
∣
1
R cos φ 
= 
1
cos φ
{
1 if |φ| ≤ φ
0
σ else
}
s
M 
=
∣
∣
∣
∣
∂h
∂φ
∣
∣
∣
∣
∣
∣
∣
∣
∂p
∂φ
∣
∣
∣
∣
−1
=
∣
∣
∣
∣
∂h
∂φ
∣
∣
∣
∣
1
R 
= 
3π
8 
cos φ
{
1 if |φ| ≤ φ
0
1
σ
√
(λ − λ
c
)
2 16
π
2 
+ 1 else.
}
Here |(a, b)| denotes the Euclidean norm 
√
a
2 
+ b
2
.
The local area scale of a map projection at a point is the reciprocal of the ratio of the
area of an infinitesimal parallelogram on the sphere to the area of its projection on the
plane. For HEALPix this scale is
s
A 
= Jh(λ, φ)
∣
∣
∣
∣
∂p
∂λ 
× 
∂p
∂φ
∣
∣
∣
∣
−1
= 
Jh(λ, φ)
|∂p/∂λ| |∂p/∂φ| 
= 
(3π/8)R
2 
cos φ
(R cos φ)R 
= 
3π
8 
,
where Jh(λ, φ) denotes the Jacobian determinant of h(λ, φ). Since a map projection is
equiareal if and only if s
A 
is constant, we see that the HEALPix map projection is indeed
equiareal.
HEALPix preserves local angles on the sphere (is conformal) at points satisfying s
P 
=
s
M 
and ∂h/∂λ · ∂h/∂φ = 0, namely at points with latitude φ = ± arccos(
√
8/(3π)) ≈
±22.88
◦
.
Under a general map projection an infinitesimal circle on the sphere centered at a given
point projects to an infinitesimal ellipse on the plane, called the Tissot ellipse of the point.
The major and minor radii, A and B respectively, of the Tissot ellipse are related to the
local scales above by two theorems of Apollonius on conjugate ellipse diameters:
s
2
P 
+ s
2
M 
= A
2 
+ B
2
, s
A 
= AB.
Thus
A =
√
s
2
P 
+ s
2
M 
+ 2s
A 
+
√
s
2
P 
+ s
2
M 
− 2s
A
B =
√
s
2
P 
+ s
2
M 
+ 2s
A 
−
√
s
2
P 
+ s
2
M 
− 2s
A
.
The radii of the Tissot ellipse at a point can be used to quantify a projection’s local
maximum angular distortion, local linear distortion, and local areal distortion at the
point via the quantities
(?) 2 arcsin
(
A − B
A + B
)
, 
A
B 
, AB,
respectively. Half of the first quantity is the greatest angular change a pair of perpendic-
ular lines meeting at the given point on the ellipsoid undergo when projected onto the
plane. For a conformal projection, this quantity is 0 (no angular distortion) at all points.
The second quantity is the aspect ratio of the Tissot ellipse. For a conformal projection,
this quantity is 1 (no linear distortion) at all points. The third quantity is the area of the
bounding rectangle of the Tissot ellipse and also the local area scale. For an equiareal pro-
jection, this quantity is a single constant at all points. For an area-preserving projection,
this quantity is 1 (no areal distortion) at all points.
13

Thus, reasonable measures of the global maximum angular distortion, global linear
distortion, and global areal distortion of a map projection are the mean, standard devia-
tion, minimum, maximum and median of the respective quantities (?). Table 1 presents
sampling estimates of these measures for HEALPix and other map projections of the
sphere of Earth mean radius 6,371,000 m (cf. [GG07]).
A.3. Formulas for the Ellipsoid. Let E
2
a,b 
⊂ R
3 
denote the ellipsoid of revolution
centered at the origin with major radius a and minor radius b. For definiteness here,
assume the ellipsoid is oblate, that is, it is generated by revolving an ellipse about its
minor axis. By a map projection of the ellipsoid, we mean an injective function from
E
2
a,b 
to the plane R
2
. In cartography, points on the ellipsoid are usually described in
terms of their longitude and geodetic latitude, that is, via the bijective parametrization
̂p : D → E
2
a,b 
defined by
̂p(λ, φ) = (a cos λ cos η(φ), a sin λ cos η(φ), b sin η(φ)).
Here η is the parametric latitude of a point
§
, which can be expressed in terms of geodetic
latitude via
η(φ) =
{
arctan(
√
1 − e
2 
tan φ) if |φ| 6 = 
π
2
φ else.
Here e = 
√
1 − (b/a)
2 
is the eccentricity of the ellipsoid. Since tan η = 
√
(1 − e
2
) tan φ
we have
cos η = 
cos φ
√
1 − e
2 
sin
2 
φ
, sin η =
√
1 − e
2 
sin φ
√
1 − e
2 
sin
2 
φ
.
As in the the spherical scenario, specifying a signature function f : D → R
2 
uniquely
determines a map projection f ◦̂ p
−1 
of the ellipsoid.
As promised, we extend the HEALPix map projection to the ellipsoid E
2
a,b 
while pre-
serving its features (i)–(iv) above. We do this in a way typical to equiareal map projec-
tions [Sny87, Section 3], namely, by applying the spherical version of the projection to
the authalic sphere of the ellipsoid.
The authalic sphere S
2
R
q 
of the ellipsoid E
2
a,b 
is the unique sphere having the same
area as the ellipsoid. The ellipsoid can be transformed into its authalic sphere via the
area-preserving bijection p ◦ l ◦̂ p
−1
, where l : D → D is defined by
l(λ, φ) = (λ, β(φ)).
Here β represents authalic latitude, which can be expressed in terms of geodetic latitude
via
β(φ) = arcsin
( 
q(φ)
q(π/2)
)
q(φ) = 
(1 − e
2
) sin φ
1 − e
2 
sin
2 
φ 
− 
1 − e
2
2e 
ln
(
1 − e sin φ
1 + e sin φ
)
.
Authalic latitude is slightly smaller in magnitude than geodetic latitude, but the two
coincide at the poles and equator. The radius of the authalic sphere is R
q 
= a
√
q(π/2)/2.
§ 
The geodetic latitude φ of a point X on E
2
a,b 
is the (smallest) angle between the equatorial plane
and the line normal to E
2
a,b 
that passes through P . This line does not pass through the center of E
2
a,b
when X is off the equator and poles.
The parametric latitude η of a point X on E
2
a,b 
is the (smallest) angle between the equatorial plane
and the line normal to E
2
a,b 
that passes through X
′
, the vertical projection of X onto a surrounding
sphere of radius a. This line always passes through the center of E
2
a,b
.
14

Max angular distortion (degrees)
 
Linear distortion
 
Areal distortion
Map projection
 
Mean
 
Std
 
Min
 
Max
 
Median
 
Mean
 
Std
 
Min
 
Max
 
Median
 
Mean
 
Std
 
Min
 
Max
 
Median
Equiareal projectionsHealpix
 
15.776
 
13.066
 
0.003
 
49.299
 
9.329
 
1.358
 
0.353
 
1.0
 
2.43
 
1.177
 
1.178
 
0.0
 
1.178
 
1.178
 
1.178
rHEALPix
 
15.776
 
13.066
 
0.003
 
49.299
 
9.329
 
1.358
 
0.353
 
1.0
 
2.43
 
1.177
 
1.178
 
0.0
 
1.178
 
1.178
 
1.178
Lambert cylindrical equal area
 
30.967
 
36.047
 
0.0
 
176.947
 
16.33
 
4.726
 
55.212
 
1.0
 
6364.621
 
1.33
 
1.0
 
0.0
 
1.0
 
1.0
 
1.0
Lambert azimuthal equal area
 
49.099
 
39.746
 
0.0
 
178.885
 
38.831
 
10.842
 
213.924
 
1.0
 
30728.11
 
1.995
 
1.0
 
0.001
 
0.918
 
1.2
 
1.0
Icosohedral Snyder equal area
 
9.541
 
3.141
 
4.878
 
17.234
 
8.831
 
1.183
 
0.066
 
1.089
 
1.352
 
1.167
 
1.0
 
0.0
 
1.0
 
1.0
 
1.0
Conformal projectionsMercator
 
0.0
 
0.0
 
0.0
 
0.002
 
0.0
 
1.0
 
0.0
 
1.0
 
1.0
 
1.0
 
4.726
 
55.212
 
1.0
 
6364.621
 
1.33
Stereographic
 
0.0
 
0.0
 
0.0
 
0.012
 
0.0
 
1.0
 
0.0
 
1.0
 
1.0
 
1.0
 
49995.603
 
6008139.487
 
1.0
 
1028684349.24
 
3.979
Lambert conformal conic
 
0.0
 
0.0
 
0.0
 
0.004
 
0.0
 
1.0
 
0.0
 
1.0
 
1.0
 
1.0
 
386.307
 
35055.539
 
0.989
 
5796465.879
 
1.608
Other projectionsEquirectangular
 
16.873
 
21.612
 
0.0
 
153.663
 
8.186
 
1.547
 
1.527
 
1.0
 
79.78
 
1.153
 
1.547
 
1.527
 
1.0
 
79.78
 
1.153
Azimuthal equidistant
 
33.508
 
28.685
 
0.0
 
169.198
 
25.584
 
2.459
 
4.097
 
1.0
 
280.216
 
1.568
 
2.459
 
4.099
 
1.0
 
280.431
 
1.568
Winkel triple
 
23.283
 
14.423
 
0.176
 
143.5
 
19.414
 
1.579
 
0.631
 
1.003
 
29.469
 
1.402
 
1.052
 
0.44
 
0.818
 
22.938
 
0.97
Table 1.
 
Sample mean, standard deviation, minimum, maximum, and median for the maximum angular distortion
function 2 arcsin((
A
 
−
 
B
)(
A
 
+
 
B
)), the linear distortion function
 
A/B
, and the areal distortion function
 
AB
 
for PROJ.4
implementations of various common global map projections of the sphere of Earth mean radius 6,371,000 m. Here
 
A
 
and
B
 
denote the major and minor radii of the Tissot ellipse of a longitude-latitude point. We used a sample of 30,000 points
chosen uniformly at random from the surface of the sphere (and not simply uniformly at random from the rectangle
 
D
).
The sphere was decapitated at latitudes
 
±
89
.
5
◦
, because some of the projections above are undefined at the poles.
15

Though l is a bijection, its inverse cannot be written in closed form. To calculate
geodetic latitude φ approximately from authalic latitude β, it is common to use the
truncated series approximation
φ ≈β +
(
1
3
e
2 
+ 
31
180
e
4 
+ 
517
5040
e
6
)
sin(2β)+
( 
23
360
e
4 
+ 
251
3780
e
6
)
sin(4β) + 
761
45360
e
6 
sin(6β).
We define the signature function of the HEALPix map projection of the ellipsoid E
2
a,b
to be h
R
q 
◦ l.
A.4. Distortion of the Ellipsoid. As for the sphere, to evaluate the distortion of the
ellipsoid E
2
a,b 
under the HEALPix map projection we compute the projection’s local scales.
Let h = h
R
q 
. Because h ◦ l ◦̂ p
−1 
= (h ◦ p
−1
) ◦ (p ◦ l ◦̂ p
−1
) and the latter composite function
preserves area, the local area scale is
̂s
A 
= s
A 
· 1 = 
3π
8 
.
For the local linear scales we need the partial derivatives of̂ p, l, and h on the interior
of D:
∂̂ p
∂λ 
= (−a sin λ cos η, a cos λ cos η, 0)
∂̂ p
∂φ 
= 
dη
dφ
(−a cos λ sin η, −a sin λ sin η, b cos η)
dη
dφ 
=
√
1 − e
2
1 − e
2 
sin
2 
φ
∂l
∂λ 
= (1, 0)
∂l
∂φ 
=
(
0, 
dβ
dφ
)
dβ
dφ 
= 
|∂̂ p/∂λ| |∂̂ p/∂φ|
R
2
q 
cos β 
.
Notice that ∂̂ p/∂λ and ∂̂ p/∂φ are orthogonal since their dot product is 1. The last
equality follows from the fact that the local area scale of p ◦ l ◦̂ p
−1 
is
1 = |∂p/∂λ × ∂p/∂β| Jl(λ, φ) |∂̂ p/∂λ × ∂̂ p/∂φ|
−1
= R
2
q 
cos β(dβ/dφ) |∂̂ p/∂λ|
−1 
|∂̂ p/∂φ|
−1 
.
16

Using the derivatives above and b
2 
= a
2
(1 − e
2
) we get
̂s
P 
=
∣
∣
∣
∣
∂h
∂λ
∣
∣
∣
∣
∣
∣
∣
∣
∂l
∂λ
∣
∣
∣
∣
∣
∣
∣
∣
∂̂ p
∂λ
∣
∣
∣
∣
−1
=
∣
∣
∣
∣
∂h
∂λ
∣
∣
∣
∣
√
1 − e
2 
sin
2 
φ
a cos φ
= 
R
q
√
1 − e
2 
sin
2 
φ
a cos φ
{
1 if |β| ≤ φ
0
σ else
}̂
s
M 
=
∣
∣
∣
∣
∂h
∂β
∣
∣
∣
∣
∣
∣
∣
∣
∂l
∂φ
∣
∣
∣
∣
∣
∣
∣
∣
∂̂ p
∂φ
∣
∣
∣
∣
−1
=
∣
∣
∣
∣
∂h
∂β
∣
∣
∣
∣
dβ
dφ
∣
∣
∣
∣
∂̂ p
∂φ
∣
∣
∣
∣
−1
=
∣
∣
∣
∣
∂h
∂β
∣
∣
∣
∣
a cos φ
R
2
q 
cos β
√
1 − e
2 
sin
2 
φ
= 
3πa cos φ
8R
q
√
1 − e
2 
sin
2 
φ
{
1 if |β| ≤ φ
0
1
σ
√
(λ − λ
c
)
2 16
π
2 
+ 1 else.
}
The HEALPix map projection of the ellipsoid is conformal at points satisfyinĝ s
P 
=̂ s
M
and ∂h/∂λ · ∂h/∂φ = 0, that is, points for which (3π/8)a
2 
cos
2 
φ = R
2
q 
(1 − e
2 
sin
2 
φ), that
is, points at latitude φ ≈ ±23.10
◦
.
The formulas for the radii of the Tissot ellipses of the ellipsoid in terms of̂ s
P 
,̂ s
M 
,
and̂ s
A 
are the same as for the sphere. Table 2 presents sampling estimates of the mean,
standard deviation, minimum, maximum and median of A/B and AB on the WGS84
ellipsoid. We did not include the popular universal transverse Mercator (UTM) conformal
map projection (or collection of projections) of the WGS84 ellipsoid, because, although
it has approximately no linear or areal distortion on its domain, its domain is not really
global, being restricted to latitudes south of 84
◦
N and north of 80
◦
S.
Appendix B. The rHEALPix map projection
The rHEALPix map projection was first described in passing in [CR07, Figure 5]
and is the following simple rearrangement of the HEALPix map projection. Consider
the image of the HEALPix map projection of the sphere or ellipsoid, number its north
polar triangles 0–3 from east to west, and do the same for the south polar triangles;
see Figure 1(a). Given integers 0 ≤ n, s ≤ 3, combine the north polar triangles of the
HEALPix map projection into a square upon north polar triangle n, and combine the
south polar triangles into a square upon south polar triangle s. More specifically, rotate
north polar triangle i by (n + i) mod 4 quarter turns counterclockwise about its center
and translate it so that its former north tip lies on the north tip of triangle n, and
rotate south polar triangle i by (s + i) mod 4 quarter turns clockwise about its center
and translate it so that its former south tip lies on the south tip of triangle s. Call the
result the (n, s)-rHEALPix map projection. For example, the (1,2)-rHEALPix map
projection of the unit sphere is pictured in Figure 1(b).
Key features of the (n, s)-rHEALPix map projection are:
(i) It is an equiareal map projection with area scaling factor 3π/8 and low average
angular and linear distortion.
(ii) Parallels of latitude in the equatorial region (authalic latitudes of magnitude at
most φ
0 
:= arcsin(2/3) ≈ 41.8
◦
) project to horizontal lines and those in the polar
(nonequatorial) region project to squares.
(iii) The projection of a parallel of latitude uniformly divided is uniformly divided.
(iv) Meridians of longitude project to vertical lines in the equatorial region and lines
converging polewards in the polar region.
17

Max angular distortion (degrees)
 
Linear distortion
 
Areal distortion
Map projection
 
Mean
 
Std
 
Min
 
Max
 
Median
 
Mean
 
Std
 
Min
 
Max
 
Median
 
Mean
 
Std
 
Min
 
Max
 
Median
Equiareal projectionsHealpix
 
15.651
 
12.993
 
0.002
 
49.329
 
9.386
 
1.36
 
0.352
 
1.0
 
2.428
 
1.178
 
1.178
 
0.0
 
1.178
 
1.178
 
1.178
rHEALPix
 
15.651
 
12.993
 
0.002
 
49.329
 
9.386
 
1.36
 
0.352
 
1.0
 
2.428
 
1.178
 
1.178
 
0.0
 
1.178
 
1.178
 
1.178
Lambert cylindrical equal area
 
30.597
 
35.817
 
0.0
 
177.669
 
16.268
 
4.954
 
66.078
 
1.0
 
5961.011
 
1.335
 
1.0
 
0.0
 
1.0
 
1.0
 
1.0
Lambert azimuthal equal area
 
48.813
 
39.741
 
0.0
 
178.364
 
38.386
 
11.306
 
198.871
 
1.0
 
18182.001
 
2.032
 
1.0
 
0.001
 
0.889
 
1.132
 
1.0
Icosohedral Snyder equal area
 
9.532
 
3.136
 
4.853
 
17.283
 
8.815
 
1.183
 
0.066
 
1.088
 
1.354
 
1.166
 
1.0
 
0.0
 
1.0
 
1.0
 
1.0
Conformal projectionsMercator
 
0.0
 
0.0
 
0.0
 
0.003
 
0.0
 
1.0
 
0.0
 
1.0
 
1.0
 
1.0
 
4.954
 
66.078
 
1.0
 
5961.011
 
1.335
Stereographic
 
0.0
 
0.0
 
0.0
 
0.005
 
0.0
 
1.0
 
0.0
 
1.0
 
1.0
 
1.0
 
148489.015
 
10765150.045
 
4.0
 
1341569280.88
 
16.512
Lambert conformal conic
 
0.0
 
0.0
 
0.0
 
0.004
 
0.0
 
1.0
 
0.0
 
1.0
 
1.0
 
1.0
 
526.865
 
38526.357
 
0.989
 
5191993.371
 
1.603
Other projectionsEquirectangular
 
16.592
 
21.569
 
0.0
 
156.982
 
8.034
 
1.546
 
1.604
 
1.0
 
77.338
 
1.153
 
1.548
 
1.597
 
1.003
 
77.08
 
1.158
Azimuthal equidistant
 
11.455
 
7.31
 
0.0
 
25.657
 
10.795
 
1.234
 
0.16
 
1.0
 
1.571
 
1.21
 
1.234
 
0.161
 
1.0
 
1.58
 
1.21
Winkel triple
 
23.231
 
14.376
 
0.249
 
145.781
 
19.296
 
1.585
 
0.702
 
1.002
 
39.228
 
1.412
 
1.053
 
0.424
 
0.821
 
19.729
 
0.972
Table 2.
 
Sample mean, standard deviation, minimum, maximum, and median for the maximum angular distortion
function 2 arcsin((
A
 
−
 
B
)(
A
 
+
 
B
)), the linear distortion function
 
A/B
, and the areal distortion function
 
AB
 
for PROJ.4
implementations of various common global map projections of the WGS84 ellipsoid.
 
Here
 
A
 
and
 
B
 
denote the major
and minor radii of the Tissot ellipse of a longitude-latitude point. We used a sample of 30,000 points chosen uniformlyat random from the surface of the ellipsoid (and not simply uniformly at random from the rectangle
 
D
). The ellipsoid
was decapitated at latitudes
 
±
89
.
5
◦
, because some of the projections above are undefined at the poles.
18

(v) The entire projection image can be divided into six isomorphic squares, four in the
equatorial region and two in the polar region.
At first glance the (n, s)-rHEALPix map projection looks like the quadrilateralized
spherical cube projections of [CO75, OL76], which are also equiareal map projections
whose images can be divided into six isomorphic squares. However rHEALPix differs
from these projections. For example, the edges of the six squares become meridians and
parallels under the inverse rHEALPix projection, which is not the case for the other two
projections.
B.1. rHEALPix Formulas for the Sphere and Ellipsoid. The signature functions
of the (n, s)-rHEALPix map projection of the sphere S
2
R 
and the ellipsoid E
2
a,b 
are g
R,n,s 
:=
¯
R ◦ g
1,n,s 
◦ h
1 
and g
R
q 
,n,s 
◦ l respectively, where g
1,n,s 
is the area-preserving bijection that
rearranges the polar triangles of the HEALPix map projection by rotation and translation:
g
1,n,s
(x, y) =





M 
(c−n)
((x, y) − t
c
) + 
(
−
3π
4 
+ n
π
2 
, 
π
2
) 
if y > 
π
4
M 
−(c−s)
((x, y) − t
c
) + 
(
−
3π
4 
+ s
π
2 
, −
π
2
) 
if y < −
π
4
(x, y) else
M =
(
0 −1
1 0
)
c =
⌊
2(x + π)
π
⌋
t
c 
=
(
−
3π
4 
+ c
π
2 
, sgn(y)
π
2
)
.
The inverse of g
1,n,s 
is
g
−1
1,n,s
(x, y) =





M 
−(c−n)
((x, y) − 
(
−
3π
4 
+ n
π
2 
, 
π
2
)
) + t
c 
if y > 
π
4
M 
(c−s)
((x, y) − 
(
−
3π
4 
+ s
π
2 
, −
π
2
)
) + t
c 
if y < −
π
4
(x, y) else,
where this time the number c of the polar triangle that (x, y) lies in is defined by the
following case statement. If y > 
π
4 
, then
c =









n mod 4 if y ≤ L
1 
and y < L
2
(n + 1) mod 4 if y < L
1 
and y ≥ L
2
(n + 2) mod 4 if y ≥ L
1 
and y > L
2
(n + 3) mod 4 else,
where L
1 
and L
2 
are the y coordinates of the diagonals of the north polar square:
L
1 
= x + 
3π
4 
− (n − 1)
π
2
L
2 
= −x − 
3π
4 
+ (n + 1)
π
2 
;
19

if y < −
π
4 
, then
c =









s mod 4 if y > L
1 
and y ≥ L
2
(s + 1) mod 4 if y ≤ L
1 
and y > L
2
(s + 2) mod 4 if y < L
1 
and y ≤ L
2
(s + 3) mod 4 else,
where L
1 
and L
2 
are the y coordinates of the diagonals of the south polar square:
L
1 
= x + 
3π
4 
− (s + 1)
π
2
L
2 
= −x − 
3π
4 
+ (s − 1)
π
2 
.
Since an rHEALPix map projection is identical to the HEALPix map projection in the
equatorial region, its spherical and ellipsoidal versions are also conformal at points with
latitude ≈ ±22.88
◦ 
and ≈ ±23.10
◦
, respectively.
Since the rHEALPix and HEALPix map projections are rigid transformations of each
other, their local scales are identical at longitude-latitude points that map to the interiors
of both projection images, which is almost all points, all points off a set of measure zero
to be more precise. This fact is reflected statistically in identical entries in Tables 1 and
2 for the two projections.
References
[AS00] H. Alborzi and H. Samet, Augmenting sand with a spherical data model, International Con-
ference on Discrete Global Grids (Santa Barbara, California), 2000.
[BS95] Lev M. Bugayevskiy and John P. Snyder, Map projections: A reference manual, Taylor
Francis, 1995.
[CO75] F. K. Chan and E. M. O’Neill, Feasibility study of a quadrilateralized spherical cube earth data
base, computer sciences corporation, Tech. Report 2-75, Monterey, California: Environmental
Prediction Research Facility, 1975.
[CR07] Mark R. Calabretta and Boudewijn F. Roukema, Mapping on the healpix grid, Monthly
Notices of the Royal Astronomical Society 381 (2007), no. 2, 865–872.
[Eul78] Leonhard Euler, De repraesentatione superficiei sphaericae super plano, Acta Academiae
Scientarum Imperialis Petropolitinae (1778).
[GG07] D. M. Goldberg and J. R. Gott III, Flexion and skewness in map projections of the earth.,
Cartographica 42 (2007), 297–318.
[GHB
+
05] K. M. Gorski, E. Hivon, A. J. Banday, B. D. Wandelt, F. K. Hansen, M. Reinecke, and
M. Bartelman, Healpix – a framework for high resolution discretization, and fast analysis of
data distributed on the sphere, The Astrophysical Journal 622 (2005), 759.
[GKWS08] MJ Gregory, AJ Kimerling, D White, and K Sahr, A comparison of intercell metrics on
discrete global grid systems, Computers Environment and Urban Systems 32 (2008), 188 –
203, [Original String]:Gregory MJ, Kimerling AJ, White D, Sahr K. 2008. A comparison
of intercell metrics on discrete global grid systems. Computers, Environment and Urban
Systems 32(3):188-203.
[KSWS99] Jon A. Kimerling, Kevin Sahr, Denis White, and Lian Song, Comparing geometrical prop-
erties of global grids, Cartography and Geographic Information Science 26 (1999), no. 4,
271–287.
[Nes11] Otakar Nesvadba, Nothing to fear from ellipsoidal harmonics, European Geosciences Union
General Assembly 2011 (Vienna, Austria), 2011.
[OL76] E. M. O’Neill and R. E. Laubscher, Extended studies of the quadrilateralized spherical cube
earth data base, Tech. Report 3-76, Computer Sciences Corp Silver Spring Md System Sci-
ences Div, 1976.
[Sny87] J. P. Snyder, Map projections–a working manual, U.S. Geological Survey professional paper,
U.S. G.P.O., 1987.
20

[Sny92] , An equal-area map projection for polyhedral globes, Cartographica 29 (1992), 10–21.
[SWK03] Kevin Sahr, Denis White, and A. Jon Kimerling, Geodesic Discrete Global Grid Systems,
Cartography and Geographic Information Science 30 (2003), 121–134.
Landcare Research, Private Bag 11052, Manawatu Mail Centre, Palmerston North
4442, New Zealand
E-mail address: Gibbr@landcareresearch.co.nz
Department of Computer Science, University of Auckland, Private Bag 92019, Auck-
land 1001, New Zealand
E-mail address: raichev@cs.auckland.ac.nz
Landcare Research, Private Bag 11052, Manawatu Mail Centre, Palmerston North
4442, New Zealand
E-mail address: SpethM@landcareresearch.co.nz
