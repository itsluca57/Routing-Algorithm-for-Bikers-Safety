---
abstract: |
  In Italy, according to ISTAT data, an average of 45 accidents
  involving cyclists occurred daily, resulting in over 200 fatalities
  and tens of thousands of injuries throughout every year[@istat_2023].
  These numbers continue to increase annually: in the first nine months
  of 2025, a 12.2% increase in fatalities was recorded compared to the
  previous year[@asaps_2025]. These accidents seem to be concentrated in
  Northern Italy and, in general, in urban areas, where over 60% of them
  occurred. Within the European context, Italy's performance is also
  concerning; the country exhibits one of the highest fatality rates
  despite having one of the lowest cycling exposure levels per person,
  as shown in Figure
  [\[fig:confronto_europa\]](#fig:confronto_europa){reference-type="ref"
  reference="fig:confronto_europa"}[@european_comparison].

  To address this, we propose a safety-oriented routing algorithm that
  integrates various risk factors to optimize path safety for cyclists.
  Our results suggest a favorable trade-off: by increasing the total
  path length (up to 50%), it is possible to achieve up to a 65%
  reduction in route dangerousness within urban environments.
  Furthermore, this project incorporates historical accident data within
  the routing areas and provides a comparative analysis across different
  cities. This approach allows us to highlight critical infrastructure
  deficiencies and safety gaps, providing evidence-based insights for
  urban planning.

  <figure id="fig:confronto_europa">
  <span class="image placeholder"
  data-original-image-src="fatality_rate_vs_exposure.jpg"
  data-original-image-title="" width="0.8\linewidth"></span>
  <figcaption>Fatality rate and cycling exposure in Europe.</figcaption>
  </figure>
author:
- |
  Luca Cesco Bolla\
  `luca.cescobolla@studenti.unipd.it`
bibliography:
- sample.bib
title: Routing Algorithm to Improve Biker's Safety
---

# Introduction

Our work is built upon a framework originally developed by Clara
Espinosa Acevedo in her Master's Thesis at Universidad Carlos III de
Madrid [@acevedo_thesis]. While the original project focused on reducing
exposure to street sexual harassment (SSH), we have repurposed the
algorithm to specifically address and enhance cyclist safety.

A significant departure from the original study lies in the data source:
while the previous framework relied on synthetic data, our approach
leverages a comprehensive dashboard from Politecnico di Milano
(PoliMi)[@dashboard_polimi]. This allowed us to integrate empirical
geographical data regarding cyclist accidents from 2022-2023, providing
a realistic foundation for historical accident locations.

Unlike conventional routing algorithms that prioritize the shortest
path, our approach provides a multi-option selection. Specifically, the
system calculates and presents three distinct routes: the shortest path,
the safest path, and a balanced path that optimizes the trade-off
between travel time and safety. By analyzing the correlation between
path length and the inherent danger of specific road segments, the
algorithm allows users to choose the level of risk they are willing to
accept for their commute.

# Method

The development phase began with the configuration of a Python virtual
environment to ensure a reproducible and isolated workspace, managing
specific dependencies without affecting the host system's global
configuration. Initially, we focused on the replication of Espinosa
Acevedo's original experiments; this stage was fundamental to
understanding the modular architecture of the safety-oriented routing
engine.

A primary technical improvement involved the map generation logic. In
the original SSH project, the geographical boundaries were pre-defined
and static, which limited the algorithm's practical flexibility. To
overcome this, we implemented a dynamic generation system that leverages
the OSMnx library to programmatically fetch and build the road network
graph from OpenStreetMap data. This system automatically calculates a
bounding box based on the user-defined origin and destination points,
ensuring that the graph topology is always tailored to the specific
surrounding area of the requested path.

Following this structural update, we shifted the focus toward
cyclist-specific risk assessment. We identified several risk factors
that contribute to the overall danger of a street segment ($e$). In our
model, each segment in the road graph is assigned a Risk Score ($RS_e$),
calculated as a weighted sum of heterogeneous factors:

$$RS_e = w_1 f_1 + w_2 f_2 + w_3 f_3 + w_4 f_4 = \sum_{i=1}^{4} w_i f_i$$

Where $RS_e$ represents the total Risk Score of the street segment $e$,
$f_i$ are the normalized risk factors, and $w_i$ are the respective
weights assigned to each factor. In our model, the four parameters are
defined as follows:

-   $f_1$, Intersections penalty: Represents the density of
    intersections along the segment. Higher values increase the risk due
    to potential vehicle-cyclist conflicts.

-   $f_2$, Roundabouts penalty: Accounts for the presence of
    roundabouts, which often represent critical points for cyclist
    navigation in urban environments.

-   $f_3$, Historical incidents risk: Derived from the PoliMi
    dashboard[@dashboard_polimi], this factor weights the segment based
    on the frequency and severity of accidents recorded in 2022-2023.

-   $f_4$, Cycling lane presence: A binary factor indicating whether a
    dedicated cycling lane exists. Unlike the other factors, this acts
    as a 'mitigating factor', effectively lowering the risk score of a
    segment.

The weights $w_i$ were calibrated prioritizing historical data ($w_3$)
as the most reliable indicator of danger, while $w_4$ is assigned a
negative coefficient in our implementation to reflect the safety benefit
of dedicated infrastructure.

## Intersections penalty

To identify critical conflict points, we extracted intersection
locations directly from the dynamically generated road graph. We
assigned a specific risk level to each intersection based on its node
degree, which is the number of street segments that converge at that
single point. The underlying assumption is that a higher number of
converging roads increases the complexity of the maneuver and the
potential for vehicle-cyclist collisions.

To model the risk associated with intersections ($f_1$), we implemented
a non-linear penalty function based on the node degree. Instead of a
linear increase, we used an exponential mapping to represent the sharp
rise in cognitive load and collision probability as the number of
converging roads increases. The penalty $P(d)$ for a node with degree
$d$ is defined as follows: $$\label{eq:intersection_penalty}
P(d) = \min\left( \left( \frac{d}{d_{max}} \right)^{\alpha}, 1.0 \right)$$
where $d_{max}$ represents the maximum degree observed in the network
(used for normalization) and $\alpha$ is the sensitivity exponent (set
to $1.3$ in our experiments). This formulation ensures that low-degree
nodes receive a negligible penalty, while complex junctions (high $d$)
approach a maximum risk value of $1.0$.

An example of the effect of
[\[eq:intersection_penalty\]](#eq:intersection_penalty){reference-type="eqref"
reference="eq:intersection_penalty"} formula on intersections is shown
in Figure [2](#fig:intersection_penalty){reference-type="ref"
reference="fig:intersection_penalty"}.

<figure id="fig:intersection_penalty">
<span class="image placeholder"
data-original-image-src="intersection_penalty.png"
data-original-image-title="" width="0.6\linewidth"></span>
<figcaption>Intersection penalty applied on Padova.</figcaption>
</figure>

## Roundabout penalty

Regarding the second risk factor ($f_2$), roundabouts were identified by
filtering OpenStreetMap metadata for 'junction=roundabout' tags. Since
roundabouts are represented in OSM as a collection of multiple edges, we
aggregated these segments to calculate a single geometric centroid for
each roundabout.

The risk score for each roundabout is determined by the number of
connecting roads. To reflect the significant hazard that circular
junctions pose to cyclists, we applied a linear penalty function with a
high baseline. The penalty $R(e)$ for a roundabout with $e$ exits is
defined as: $$\label{eq:roundabout_penalty}
R(e) = \min(B + w_{round} \cdot e, 1.0)$$ Where $B$ represents the
baseline risk for any circular junction and $w_{round}$ is the weight
assigned to each additional exit. In our experimental setup, we
calibrated these parameters by setting $B = 0.4$ and $w_{round} = 0.12$.

This specific calibration ensures that a roundabout with the minimum
typical number of exits ($e=3$) starts with a high risk base of $0.76$,
reaching the maximum penalty of $1.0$ at five or more exits.

An example of the effect of
[\[eq:roundabout_penalty\]](#eq:roundabout_penalty){reference-type="eqref"
reference="eq:roundabout_penalty"} formula on roundabouts is shown in
Figure [3](#fig:roundabout_penalty){reference-type="ref"
reference="fig:roundabout_penalty"}.

<figure id="fig:roundabout_penalty">
<span class="image placeholder"
data-original-image-src="roundabout_penalty.png"
data-original-image-title="" width="0.6\linewidth"></span>
<figcaption>Roundabout penalty applied on Padova.</figcaption>
</figure>

## Crossing penalty

Since the routing algorithm operates on the graph's edges rather than
its nodes, both the intersection penalty ($f_1$) and the roundabout
penalty ($f_2$) calculated for each node must be propagated to the
adjacent street segments. We implemented an additive risk accumulation
model to handle cases where a street segment is influenced by multiple
conflict points simultaneously.

Our approach sums the individual distance-weighted penalties, this
ensures that if an edge is located within the influence radius of, for
example, two intersections and a roundabout, its total risk score
reflects the cumulative complexity of the area. This additive
contribution is particularly effective at identifying complex urban
zones where the density of junctions creates a significantly higher
cognitive load and collision risk for cyclists. To maintain consistency,
this total sum is then capped at a maximum risk value of 1.0.

We implemented a linear propagation model to distribute this risk based
on the physical distance from the junction. The influence of a conflict
point decreases linearly as the distance increases, reaching zero at a
defined threshold. Distance propagation $P_{dist}$ is defined as
follows: $$\label{eq:distance_propagation}
P_{dist}(d) = \max\left(0, 1 - \frac{d}{L}\right)$$ where $d$ represents
the distance from the point and $L$ is the influence limit fixed at 100
meters in our experimental setup.. This approach ensures that the danger
level is 'brought' onto the surrounding street segments in a linear
relation, effectively weighting the portions of the road that are
closest to the conflict point more heavily.

To ensure accurate distance calculations in meters, the graph geometry
was projected into the EPSG:3857 coordinate reference system. This
allowed the distance propagation function to operate on a metric scale,
ensuring that the 100 meter influence threshold remains consistent
across different geographical latitudes.

Using the resulting crossing penalty, we generated a heat map of the
study area to visualize the risk distribution, as shown in Figure
[4](#fig:crossing_heat_map){reference-type="ref"
reference="fig:crossing_heat_map"}.

<figure id="fig:crossing_heat_map">
<span class="image placeholder"
data-original-image-src="crossing_heat_map.png"
data-original-image-title="" width="0.7\linewidth"></span>
<figcaption>Crossing heatmap in Padova.</figcaption>
</figure>

## Historical incident risk

Integrating historical accident data ($f_3$) presented a significant
technical challenge, as the PoliMi Dashboard[@dashboard_polimi] provides
aggregated spatial visualizations rather than direct access to raw,
granular accident coordinates data that were not exposed through an API
or a downloadable format. Initially, a manual extraction of accident
locations within the city of Padua was conducted to validate the routing
algorithm. Although this approach was sufficient for local testing, it
lacked the scalability required for a system intended to operate
throughout the national territory.

The only way to access the locations of single points was through the
interactive 2022-2023 incident map provided as part of the dashboard
interface. This visual representation, while informative for human
users, required a programmatic approach to be translated into a format
compatible with our routing engine.

To automate this extraction, we utilized the Selenium framework. The
system was designed to programmatically navigate the dashboard, center
the map in the specific municipality requested by the user, and capture
the visual data.

A significant technical hurdle encountered during this process was the
low graphical quality of incident markers when capturing the entire
municipality map. Because the zoom level required to encompass the full
urban area is significantly zoomed-out, the resolution of individual
incident points degrades. This leads to spatial aliasing and the
overlapping of markers, making it difficult to distinguish between
single accidents and dense clusters.

### Image Stitching

To mitigate the resolution loss, we implemented a tiling and stitching
strategy. Instead of capturing a single low-resolution overview, the
Selenium script was programmed to divide the municipality area into four
high-resolution quadrants. To reconstruct the full map without spatial
discontinuities, we developed a custom stitching algorithm based on
Feature Matching.

The process utilizes the ORB algorithm to detect key visual features in
the overlapping regions of the quadrants. By matching these features and
applying RANSAC (Random Sample Consensus) robust estimation method, the
system calculates the precise pixel translation offsets ($dx, dy$)
between the images. This approach effectively eliminates stitching
errors caused by slight alignment variations, resulting in a single
high-fidelity composite image shown in Figure
[5](#fig:incidents_map){reference-type="ref"
reference="fig:incidents_map"}. This final \"canvas\" preserves a higher
clarity of the incident markers, enabling far more accurate extraction
of historical risk data.

<figure id="fig:incidents_map">
<span class="image placeholder"
data-original-image-src="incident_map.png" data-original-image-title=""
width="0.7\linewidth"></span>
<figcaption>Incidents map of Padova.</figcaption>
</figure>

### Incident Point Detection via Computer Vision

Once the high-resolution composite map is generated, we employ a
multi-stage Computer Vision pipeline to extract the pixel coordinates of
the incident markers. This process is designed to isolate the specific
color signatures used by the dashboard to represent accidents (e.g.,
injuries and fatalities) and accurately determine their center points.
The methodology follows these steps:

1.  **Color Space Transformation:** The composite image is converted
    from the standard BGR color space to the HSV (Hue, Saturation,
    Value) space. Unlike BGR, the HSV space is more robust to variations
    in lighting and pixel intensity, making it ideal for isolating
    specific colors.

2.  **Red Chromatic Masking:** We define two distinct ranges in the HSV
    spectrum to capture the red hues associated with incident markers.
    This is necessary because the red hue in the HSV space wraps around
    the $0^\circ$--$180^\circ$ range (appearing at both the very
    beginning and the end of the spectrum). By applying a bitwise OR
    operation between these two masks, we generate a binary mask where
    the incident markers are isolated from the map background.

3.  **Geometric Feature Extraction:** To distinguish actual markers from
    background noise and precisely locate their centroids, we apply the
    Hough Circle Transform on the binary mask. The algorithm is tuned
    with a high sensitivity and a small radius to detect the small,
    circular dots identifying the accidents.

The resulting list of pixel coordinates $(cx, cy)$ represents the exact
location of each historical incident on the high-fidelity canvas.

But to translate the pixel positions of detected incidents into
geographical coordinates (Latitude and Longitude), two fundamental
parameters were required: a spatial scale and an anchor point.

### Pixel-to-Meters Scaling

To accurately map the incident markers from the composite image onto the
geographical graph, we established a conversion factor between pixels
and meters. As shown in the bottom-right corner of Figure
[5](#fig:incidents_map){reference-type="ref"
reference="fig:incidents_map"}, the dashboard includes a dynamic scale
bar. We developed an automated pipeline to extract this information
using Computer Vision and Optical Character Recognition (OCR).The
methodology consists of two steps:

1.  **Scale Bar Detection:** Using the OpenCV library, we isolate the
    scale area and apply a binary threshold. We then perform contour
    detection to identify the scale bar. The algorithm filters
    candidates based on their aspect ratio ($w/h$) and area to
    distinguish the horizontal line of the scale from background noise
    or text. This process obtains the exact length of the bar in pixels
    ($L_{px}$).

2.  **OCR Numerical Extraction:** A specific sub-region containing the
    scale text is cropped and pre-processed. We utilize the PaddleOCR
    framework to recognize the numerical value and the unit of
    measurement. A regular expression is then used to parse this string,
    converting all units into a standard metric value ($V_{m}$).

To convert pixel distances into metric units, we calculate a spatial
resolution ratio. Based on the values extracted from the map's scale
bar, the ratio representing how many meters a single pixel covers is
defined as: $$\label{eq:pixel_scale}
R_{m/px} = \frac{V_{m}}{L_{px}}$$ where $V_{m}$ is the real-world
distance scale recognized via OCR and $L_{px}$ is the measured length of
the scale bar in pixels. Consequently, any spatial distance $d_{px}$
identified on the incident map is transformed into its metric equivalent
$d_{m}$ by the simple linear transformation:
$d_{m} = d_{px} \cdot R_{m/px}$.

### Anchor Point

To translate pixel positions into geographic coordinates, a 'bridge'
between the image space and the real-world coordinate system was
required. Since the dashboard map did not expose its internal metadata
(such as the bounding box or center coordinates) it was necessary to
identify at least one anchor point for which both the pixel coordinates
and the geographic coordinates were known. By establishing this
reference point and combining it with the previously obtained spatial
scale, we were able to calculate the real-world coordinates of each
detected accident through a linear transformation of the pixel
displacement.

To establish a reliable spatial reference, we developed an automated
image registration pipeline that aligns the dashboard's incidents map
with the georeferenced dynamically generated routing map retrieved from
OpenStreetMap (explained in section 2). This process identifies the
exact pixel coordinates corresponding to the top-left corner of the
routing map, the geographical coordinates of which are already
established. The methodology utilizes the following Computer Vision
techniques:

1.  **SIFT Feature Extraction:** We employ the Scale-Invariant Feature
    Transform (SIFT) algorithm to detect distinctive keypoints in both
    images. SIFT is chosen for its robustness against differences in
    scale and rotation, ensuring that street intersections and landmarks
    are recognized even if the two maps have different scales.

2.  **Feature Matching and Ratio Test:** The detected keypoints are
    compared using a K-Nearest Neighbors matcher. To filter out false
    correspondences, we apply Lowe's Ratio Test, which only accepts
    matches where the distance to the closest neighbor is significantly
    smaller (70%) than the distance to the second closest.

3.  **Homography Estimation:** Using the set of validated matches, we
    calculate the Homography Matrix ($H$) via the RANSAC algorithm. This
    matrix represents the perspective transformation required to project
    a point from the dashboard map's space to the routing map's
    coordinate.

By applying the perspective transformation to the origin point $(0,0)$
of the georeferenced routing map, we obtain the precise pixel
coordinates of the Anchor Point on the dashboard map.

### Final Coordinate Projection

The final step of the historical risk integration is the conversion of
pixel displacements into geographic coordinates. Since the Earth's
surface is not a flat plane, the translation from metric distances to
decimal degrees requires adjusting for latitude, as the distance between
longitudes decreases as one moves toward the poles.

Using the Anchor Point as the origin $(West, North)$, the algorithm
calculates the longitudinal and latitudinal offsets
($\Delta\lambda, \Delta\phi$) for each incident $P_i$ as follows:
$$\label{eq:delta_lambda}
\Delta\lambda = \frac{dx \cdot R_{m/px}}{111.412 \cdot \cos(\theta_m)}$$
$$\label{eq:delta_phi}
\Delta\phi = \frac{-(dy \cdot R_{m/px})}{111.132,95}$$ where $dx$ and
$dy$ are the pixel distances from the anchor, $R_{m/px}$ is the spatial
scale, and $\theta_m$ is the medium latitude of the area. The constant
values used in Equations
[\[eq:delta_lambda\]](#eq:delta_lambda){reference-type="ref"
reference="eq:delta_lambda"} and
[\[eq:delta_phi\]](#eq:delta_phi){reference-type="ref"
reference="eq:delta_phi"} represent the physical length of one degree of
longitude and latitude on the Earth's surface, based on the WGS84 model.

This processed dataset effectively populates the road graph with
geolocated historical risk nodes, allowing the routing algorithm to
evaluate the safety of each segment based on evidence of past accidents.

## Cycle lane presence

The final component of our risk model $f_4$, accounts for the presence
of dedicated cycling infrastructure. Unlike the previous factors that
increase the perceived risk, the presence of a bike lane acts as a
safety buffer, significantly reducing the overall cost of a route.To
isolate these safe paths, we implemented a topological subtraction
method using the OSMnx library.

We retrieve two distinct graphs for the same municipality: a driving
graph $G_{drive}$, containing all roads accessible by motorized
vehicles, and a cycling graph $G_{bike}$, which includes all paths
accessible to bicycles.

By calculating the difference between these two sets of edges, we are
able to identify segments that are exclusively traveled by bicycles,
such as physically separated bike lanes, park paths, and restricted
cycling tracks.

These safespots in our routing algorithm are assigned a negative risk
weight, effectively incentivizing the selection of protected routes even
when they do not represent the shortest path.

An example of cycling lane found in a graph is shown in Figure
[6](#fig:cycling_lane_map){reference-type="ref"
reference="fig:cycling_lane_map"}.

<figure id="fig:cycling_lane_map">
<span class="image placeholder"
data-original-image-src="cycle_lanes.png" data-original-image-title=""
width="0.7\linewidth"></span>
<figcaption>Cycle lanes in Padova.</figcaption>
</figure>

# Routing

Once all individual risk factors were obtained, we integrated them into
a unified cost function. Each factor was assigned a specific weight
($w_i$) reflecting its relative impact on cyclist safety.

In our experimental setup, these weights were established as
$w_{cross} = 0.4$ for the structural penalty of intersections and
roundabouts ($f_1 + f_2$), $w_{hist} = 1.2$ for the empirical risk
derived from historical accidents ($f_3$), and $w_{cycle} = -0.6$ for
the presence of dedicated cycling infrastructure ($f_4$). The negative
value assigned to the cycle lane factor acts as a safety bonus, reducing
the total arc cost and incentivizing the selection of protected routes.

The user is then provided with different navigation options:

1.  Shortest Path: Minimizes the physical distance ($L$) of the route.

2.  Safest Path: Minimizes the combined cost of the four risk factors
    ($f_1, f_2, f_3, f_4$).

3.  Balanced Path: Minimizes a hybrid cost function where specific
    weights are assigned to both distance and safety. In our
    experiments, we assigned a weight of 0.3 to length and 0.7 to
    safety, calculating the path based on the minimization of the value:
    $0.3 \cdot L + 0.7 \cdot C_{safety}$.

<figure id="fig:routing">
<span class="image placeholder"
data-original-image-src="shortest_safest_balanced.png"
data-original-image-title="" width="0.7\linewidth"></span>
<figcaption>Calculated route example.</figcaption>
</figure>

# Analysis

## Municipality overview

The initial phase of the research focused on extracting all available
data from the dashboard[@dashboard_polimi]. The incident points provided
by the platform were filterable based on the type of vehicles involved
in the accident, the cyclist's gender, and the severity of the injury
(categorized as either non-fatal injury or death).

By repeatedly executing the selenium procedure to get points coordinates
with different filter sets, we were able to isolate specific subsets of
incidents. Each newly detected point was referenced with the existing
dataset: when a coordinate match was identified, the corresponding
record was enriched with a new attribute representing the filter type,
thereby assigning specific values to each incident point based on the
active filter.

Using the data collected for each municipality, we were able to generate
visualizations to provide an overview of the local safety situation. An
example of this analysis for the city of Padova is presented in Figure
[8](#fig:municipality_analysis){reference-type="ref"
reference="fig:municipality_analysis"}.

<figure id="fig:municipality_analysis">
<span class="image placeholder"
data-original-image-src="incidents_analysis_Padova.png"
data-original-image-title="" width="1\linewidth"></span>
<figcaption>Municipality analysis of Padova.</figcaption>
</figure>

## Municipalities comparison

The subsequent step involved normalizing the number of accidents per
10,000 inhabitants for the selected municipality, population data for
each municipality was obtained from ISTAT demographic
studies[@istat_populations]. This normalization was essential for a
meaningful comparative analysis, allowing us to benchmark the local data
against major Italian cities and other provinces within the same region.
Consequently, we were able to contextualize the municipality's safety
performance within a broader regional and national framework.

The comparison with major Italian cities is shown in Figure
[9](#fig:big_cities_comparison){reference-type="ref"
reference="fig:big_cities_comparison"} and the one with other provinces
in Figure [10](#fig:provinces_comparison){reference-type="ref"
reference="fig:provinces_comparison"}.

## Registered cars and normalized incidents

Furthermore, we integrated data regarding the number of registered
automobiles for each province within the target region, sourced from
open data provided by the Ministry of Infrastructure and
Transport[@registered_vehicles]. This allowed us to construct a
comparative graph correlating the density of registered vehicles with
the normalized number of accidents per 10,000 inhabitants. By analyzing
these two variables together, we aimed to identify whether a higher
concentration of motor vehicles directly corresponds to an increased
incidence of bicycle-related accidents at the provincial level.

An example of this study for Veneto region is shown in Figure
[11](#fig:registered_vehicles){reference-type="ref"
reference="fig:registered_vehicles"}.

## Density and Incident Rate scatter

To further refine our analysis, we extracted the land area for each
municipality using OpenStreetMap (OSM) data. This allowed us to
calculate the population density (inhabitants per $km^2$) for each
territory and compare it with the normalized incident rate per 10,000
inhabitants. By correlating demographic density with accident frequency,
we aimed to investigate whether a more compact urban environment,
characterized by higher human concentration and potentially more complex
traffic dynamics, acts as a significant driver for bicycle-related
risks.

We applied this study to all municipalities involved in our previous
graphs in Figure [12](#fig:scatter){reference-type="ref"
reference="fig:scatter"}.

<figure id="fig:big_cities_comparison">
<span class="image placeholder"
data-original-image-src="normalized_incident_big_cities_comparison.png"
data-original-image-title="" width="0.77\linewidth"></span>
<figcaption>Big cities comparison of Padova.</figcaption>
</figure>

<figure id="fig:provinces_comparison">
<span class="image placeholder"
data-original-image-src="normalized_incident_province_comparison.png"
data-original-image-title="" width="0.77\linewidth"></span>
<figcaption>Provinces comparison of Padova.</figcaption>
</figure>

<figure id="fig:registered_vehicles">
<span class="image placeholder"
data-original-image-src="cars_incidents_ratio.png"
data-original-image-title="" width="0.8\linewidth"></span>
<figcaption>Registered cars / incidents per 10,000 inhabitants
ratio.</figcaption>
</figure>

<figure id="fig:scatter">
<span class="image placeholder"
data-original-image-src="density_vs_incidents_scatter.png"
data-original-image-title="" width="0.8\linewidth"></span>
<figcaption>Density / normalized incidents scatter</figcaption>
</figure>

# Conclusion

This research presented a safety-oriented routing algorithm designed to
mitigate the risks faced by cyclists in urban environments. By
integrating heterogeneous risk factors---such as junction complexity,
historical accident data extracted via computer vision, and the presence
of dedicated infrastructure. We moved beyond traditional shortest-path
logic toward a more human-centric navigation model.

Our experimental results, particularly in the case study of Padova,
demonstrate that a significant increase in safety is achievable with a
manageable impact on travel time. Specifically, we observed that by
accepting a route up to 50% longer, cyclists can reduce their exposure
to high-risk segments by up to 65%

Technically, this study validated the effectiveness of using Computer
Vision and automated scraping to bridge the gap between web dashboards
and dynamic routing applications. The successful georeferencing of
historical incidents via Selenium and image registration demonstrates
that routing is possible even when granular datasets are not directly
accessible.

Future developments could involve the integration of real-time data,
such as weather conditions and traffic flow, to provide a dynamic risk
assessment. Additionally, expanding the model to include crowdsourced
feedback would allow the algorithm to account for perceived temporary
hazards, further refining the decision-making process for urban
commuters.
