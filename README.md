# Sound Object Analyzer v3.6

## Technical Documentation

### Overview

The Sound Object Analyzer processes participant drawings of perceived sound objects, extracting geometric representations suitable for quantitative analysis. The tool generates composite visualizations by frequency condition and computes average contours representing the central tendency of spatial representations across participants.

---

### Pipeline Overview

The complete analysis workflow proceeds through the following stages:

**1. Data Collection** (External to this tool)

Participants draw perceived sound object shapes on a standardized 20x20 unit grid using a drawing interface. Each trial captures two shapes (red for diotic, blue for dichotic conditions) along with participant-indicated centroids. Drawing data including area measurements and centroid coordinates are recorded in a Google Sheets spreadsheet.

**2. Data Export**

Participant drawings are exported as PNG images with standardized filenames encoding participant ID, trial number, and frequency condition.

**3. Ground Truth Retrieval** (Optional)

The analyzer connects to Google Sheets via a read-only Apps Script API to retrieve participant-recorded centroid coordinates. These ground truth values take precedence over algorithmically computed centroids when available.

**4. Image Processing**

For each PNG image:
- Color classification separates red and blue shapes
- Connected component analysis isolates the primary shape from centroid markers
- Radial contour extraction produces angle-aligned boundary representations
- Centroid markers are detected and removed from overlay pixels

**5. Composite Generation**

Individual shapes are grouped by frequency condition and composited:
- Semi-transparent overlays visualize shape distribution
- Point-wise averaging of angle-aligned contours produces mean boundaries
- Gaussian smoothing reduces high-frequency noise in averaged contours

**6. Output**

The tool produces composite PNG images for each frequency condition displaying overlaid shapes, average contours, and mean centroids with associated metadata.

---

### Google Sheets Integration

#### Purpose

The Google Sheets connection provides ground truth centroid coordinates recorded by participants during the drawing task. These coordinates represent the participant's intended sound source location, which may differ from the geometric center of the drawn shape.

#### Architecture

A Google Apps Script deployed as a web application serves as a read-only REST API. The script accesses the spreadsheet containing experimental data and returns JSON-formatted records.

#### Spreadsheet Structure

The expected column layout:

| Column | Field | Description |
|--------|-------|-------------|
| A | Timestamp | Recording timestamp |
| B | Participant | Participant identifier |
| C | Response Number | Trial number |
| D | Frequency | Stimulus frequency in Hz |
| E | SPL | Sound pressure level in dB |
| F | Shape Color | Condition indicator (red/blue) |
| G | Area | Shape area in grid squares (units squared) |
| H | Centroid X | X-coordinate in unit space (-10 to +10) |
| I | Centroid Y | Y-coordinate in unit space (-10 to +10) |

#### API Deployment

The Apps Script requires configuration of the target spreadsheet ID and sheet name. Once deployed as a web application with public access, the script URL is entered into the Sound Object Analyzer interface.

#### Data Flow

1. The analyzer sends an HTTP GET request to the Apps Script URL
2. The script reads all records from the configured spreadsheet (read-only operation)
3. Records are returned as a JSON array containing participant, trial, frequency, color, area, and centroid fields
4. The analyzer matches records to uploaded images by participant ID, trial number, frequency, and color
5. Matched centroid coordinates are used as the reference point for radial contour extraction

#### Record Matching

For each processed image, the analyzer attempts to locate a corresponding spreadsheet record using the following criteria:
- Participant identifier (case-insensitive string match)
- Trial number (exact integer match)
- Frequency (within 1 Hz tolerance)
- Color (exact string match: "red" or "blue")

When a match is found, the spreadsheet centroid coordinates are converted from unit space to pixel coordinates and used for all radial operations. Unmatched images use calculated centroids derived from the shape's geometric center.

---

### Input Specifications

**Image Format**: PNG files at 1000x1000 pixel resolution

**Naming Convention**: `{ParticipantID}_{TrialNumber}_{Frequency}Hz.png`

**Coordinate System**: Images map to a Cartesian plane spanning -10 to +10 units on both axes, with the origin at image center (pixel 500, 500). The scale factor is 50 pixels per unit.

**Color Encoding**: Each image contains two shapes representing different binaural phase conditions:
- Red shapes: Diotic condition (interaural phase difference = 0)
- Blue shapes: Dichotic condition (interaural phase difference = pi)

---

### Processing Pipeline

#### 1. Color Classification

Pixels undergo classification into discrete categories using a two-stage approach combining HSL (Hue, Saturation, Lightness) color space analysis with supplementary RGB threshold checks.

**HSL-Based Classification**

The HSL transformation follows the standard formulation where hue is computed as a value from 0 to 360 degrees, saturation and lightness as percentages from 0 to 100.

Primary classification thresholds:
- **White**: Lightness > 95%, or saturation < 8% with lightness > 85%
- **Black**: Lightness < 5%, or saturation < 8% with lightness < 15%
- **Gray**: Saturation < 6%
- **Red**: Hue in [320, 360] or [0, 40], saturation >= 8%, lightness in [5, 95]
- **Blue**: Hue in [170, 240], saturation >= 8%, lightness in [5, 95]
- **Purple**: Hue in (240, 350), saturation >= 8%, lightness in [8, 92]

**RGB-Based Supplementary Classification**

To capture edge cases where HSL classification may fail (particularly for dark pixels affected by gridline overlap), additional RGB-based rules apply:

- Purple detection: R > 40, B > 40, G < 0.9 * max(R, B), with min(R, B) / max(R, B) > 0.35
- Red detection: R > 50, R > 1.2 * G, R > 1.1 * B
- Blue detection: B > 50, B > 1.1 * G, B > 1.1 * R
- Dark colored pixels (lightness < 30%): Assigned to the dominant RGB channel

**Purple Pixel Handling**

Purple pixels, which occur in regions where red and blue shapes overlap, are assigned to both color sets. This ensures that neither shape's contour is truncated at overlap regions.

#### 2. Connected Component Analysis

Following color classification, flood fill analysis identifies connected components within each color class using 8-connectivity (including diagonal neighbors). The algorithm employs a stack-based iterative approach to avoid recursion depth limitations.

Components smaller than 30 pixels are discarded as noise. The largest component for each color is designated as the primary shape; smaller components (typically centroid markers placed by participants) are excluded from contour analysis.

#### 3. Centroid Determination

Shape centroids are determined by one of two methods:

**Ground Truth Centroids** (when Google Sheets data is connected): Centroid coordinates are retrieved from an external spreadsheet containing participant-recorded positions. These coordinates, specified in unit space, are converted to pixel coordinates for subsequent radial sampling.

**Calculated Centroids** (default): The arithmetic mean of all pixel coordinates within the largest connected component:

```
centroid_x = (1/N) * sum(x_i)
centroid_y = (1/N) * sum(y_i)
```

where N is the pixel count and (x_i, y_i) are individual pixel coordinates.

#### 4. Gap Filling

To address potential discontinuities in shape boundaries (arising from drawing artifacts or color overlap regions), a radial gap detection algorithm identifies and fills missing segments.

**Gap Detection**

The shape boundary is analyzed at 720 angular positions (0.5 degree resolution). For each angle, the algorithm identifies the maximum radial distance from centroid to any pixel of the same color within an angular tolerance of 0.75 degrees.

Gaps are defined as angular positions where either:
- No pixels exist within the angular tolerance (null points)
- The radial distance is less than 30% of the median radial distance across all valid angles

**Gap Filling**

For each contiguous gap region, the algorithm draws a straight line connecting the valid boundary points immediately preceding and following the gap. This linear interpolation preserves shape continuity without introducing artificial curvature.

#### 5. Boundary Extraction

Boundary pixels are identified as shape pixels having at least one 8-connected neighbor that does not belong to the shape. This produces a single-pixel-wide outline of the filled shape.

#### 6. Radial Contour Extraction

The contour is represented as a sequence of points sampled at uniform angular intervals from the centroid. This angular parameterization enables direct comparison and averaging across shapes of different sizes and aspect ratios.

**Sampling Procedure**

For each of N angular positions (default N = 1000, corresponding to 0.36 degree resolution):

1. Compute the target angle: theta_i = (2 * pi * i) / N
2. Identify all boundary pixels within angular tolerance (1.5 times the angular step)
3. Select the boundary pixel at maximum radial distance from centroid
4. Convert pixel coordinates to unit space coordinates

**Truncation Handling**

Points with radial distances below 25% of the median distance are marked as potentially truncated. These points are replaced via linear interpolation between the nearest valid points on either side of the truncated region.

#### 7. Centroid Marker Removal

Participant drawings typically include a cross-shaped (+) marker indicating the perceived centroid. This marker must be removed before generating composite overlays to prevent visual clutter.

**Marker Detection**

The algorithm scans radially outward from the calculated centroid at 1-degree angular increments, identifying continuous lines of pixels (arm detection). Two perpendicular arms (within 80-100 degrees of each other) with combined length exceeding 10 pixels indicate the presence of a cross marker.

**Selective Removal**

Marker pixels within the detected radius are removed, except those coinciding with the shape boundary. This preservation prevents gaps in the contour where the marker intersects the shape edge.

---

### Composite Generation

#### Overlay Composition

For each frequency condition, individual shape overlays are composited using alpha blending. Each shape is rendered with user-specified opacity (default 5%), allowing visualization of shape distribution density.

The compositing formula for each pixel:

```
output_alpha = 1 - (1 - alpha_1) * (1 - alpha_2) * ... * (1 - alpha_n)
output_color = sum(color_i * alpha_i * product(1 - alpha_j for j < i)) / output_alpha
```

This over-composition method produces darker regions where more shapes overlap, providing immediate visual feedback on the spatial consistency of representations.

#### Average Contour Computation

The average contour represents the central tendency of shape boundaries across participants.

**Centroid Alignment**

Each contour is translated such that its centroid lies at the origin:

```
contour_centered[i] = contour[i] - centroid
```

**Point-wise Averaging**

Because contours share the same angular parameterization, corresponding points can be averaged directly by index:

```
average_contour[i] = (1/M) * sum(contour_j_centered[i]) + mean_centroid
```

where M is the number of valid shapes and mean_centroid is the arithmetic mean of all shape centroids.

**Gaussian Smoothing**

The averaged contour undergoes iterative smoothing using a discrete Gaussian kernel approximation with weights [1, 4, 6, 4, 1] (normalized sum = 16). Each iteration updates point positions as:

```
smoothed[i] = original[i] + strength * (gaussian_weighted_mean[i] - original[i])
```

Default parameters: 3 iterations at 0.5 strength. Smoothing reduces high-frequency noise while preserving overall shape characteristics.

---

### Output Specifications

**Composite Images**: PNG files at 1000x1000 pixels containing:
- Coordinate grid with axis labels
- Reference circle at radius 3 units
- Semi-transparent overlay of all individual shapes
- Average contour line (4-pixel stroke width)
- Mean centroid markers

**File Naming**: `composite_{frequency}Hz.png`

**Metadata Display**: Each composite includes text indicating:
- Sample count per color (N)
- Mean centroid coordinates
- Centroid shift distance between conditions

---

### Parameter Reference

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| Radial Samples | 1000 | 1000-5000 | Angular sampling resolution for contour extraction |
| Overlay Opacity | 5% | 5-50% | Alpha value for individual shape overlays |
| Contour Smoothing | 3 | 0-10 | Gaussian smoothing iterations for average contour |

**Angular Resolution**: At 1000 samples, contour points are spaced at 0.36 degrees. Increasing to 5000 samples provides 0.072 degree resolution, beneficial for shapes with fine angular detail.

---

### Methodological Considerations

**Color Overlap Regions**

The assignment of purple pixels to both red and blue sets ensures that neither condition's contour is artificially truncated where shapes overlap. This is particularly relevant for low-frequency conditions where sound object representations may have substantial spatial overlap.

**Radial vs. Cartesian Sampling**

Traditional contour extraction methods (e.g., marching squares) produce perimeter-ordered point sequences unsuitable for direct averaging, as corresponding anatomical features may appear at different indices. Radial sampling from a common reference point (centroid) ensures that points at the same angular position across shapes represent comparable spatial directions, enabling meaningful point-wise averaging.

**Centroid Source Priority**

When ground truth centroids are available from external data sources, these take precedence over calculated centroids. Calculated centroids represent the geometric center of the drawn shape, which may differ from the participant's intended sound source location, particularly for asymmetric shapes.

---

### Google Apps Script Deployment

#### Installation Procedure

1. Navigate to https://script.google.com and create a new project
2. Replace the default code with the provided Apps Script
3. Configure the SHEET_ID constant with the target spreadsheet identifier (found in the spreadsheet URL between /d/ and /edit)
4. Configure SHEET_NAME if the data tab has a non-default name
5. Save the project
6. Execute the testAPI() function to verify configuration and authorize access
7. Deploy as a web application:
   - Execute as: Script owner
   - Access: Anyone
8. Copy the deployment URL for use in the Sound Object Analyzer

#### Security Considerations

The Apps Script operates in read-only mode. The script uses only getValues() for data retrieval and contains no write operations (setValue, setValues, insertRows, deleteRows, or similar). The spreadsheet data cannot be modified through the API.

The web application deployment with "Anyone" access allows the browser-based analyzer to retrieve data without authentication. The script accesses only the specifically configured spreadsheet; no other Google account data is exposed.

---

### Dependencies

- JSZip 3.10.1: ZIP archive extraction
- FileSaver.js 2.0.5: Client-side file download
- Tailwind CSS: Interface styling

---

### Version History

**v3.6**: Color-specific radial tracing with purple pixel dual-assignment; conservative gap fill thresholds; centroid marker detection and removal
