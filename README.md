flowchart LR
  %% Global OUT_ROOT context
  classDef artifact fill:#f9f9f9,stroke:#bbb,color:#333,stroke-width:1px;
  classDef io fill:#eef7ff,stroke:#70a1ff,color:#103a6e,stroke-width:1px;
  classDef step fill:#fff7e6,stroke:#f0b429,color:#5a3b00,stroke-width:1px;

  %% Inputs
  A1[LIS2 TestSummaryLog\n(site, part, date window)]:::io
  A2[LCVIS anapy itemlists\n(product_ids, lcos_cell_sn, um_per_cpx, corners)]:::io

  %% Swimlane 1: match.py
  subgraph S1[match.py]
    direction TB
    M1[Filter LIS2 prod tests\nby PART & LOOKBACK]:::step
    M2[Fetch LCVIS itemlists\n(via anapy)]:::step
    M3[Normalize serials &\nfind common keys]:::step
    M4[Download matched LCVIS\nall_defects.xlsx blobs]:::step

    Mout1[LIS2_{PART}_tests.csv]:::artifact
    Mout2[LCVIS_{PART}_itemlists.csv]:::artifact
    Mout3[{PART}_matching_serials.csv\n(Key, LIS2_Serial, LCVIS_Serial)]:::artifact
    Mout4[Downloaded folder tree\n(all_defects*.xlsx per unit)]:::artifact

    A1 --> M1 --> Mout1
    A2 --> M2 --> Mout2
    M1 --> M3
    M2 --> M3 --> Mout3
    Mout3 --> M4 --> Mout4
  end

  %% Swimlane 2: newtst.py
  subgraph S2[newtst.py]
    direction TB
    N1[Aggregate LIS2 combined_defects\nfrom archive paths]:::step
    N2[Parse LCVIS all_defects.xlsx\n+ CoordConverter (LCVIS→LIS2)]:::step
    N3[Plane filter; per-serial\nDistanceMerge with radius escalation\n(50→75 µm)]:::step
    N4[Write detailed one-to-one pairs\n(+ metrics & plots)]:::step

    Nin1[LIS2_{PART}_tests.csv]:::artifact
    Nin2[LCVIS_{PART}_itemlists.csv]:::artifact
    Nin3[{PART}_matching_serials.csv]:::artifact
    Nin4[all_defects*.xlsx]:::artifact

    Nout1[LIS2_{PART}_matched_defects.csv]:::artifact
    Nout2[matched_pairs_by_distance.csv]:::artifact
    Nout3[matched_pairs_detailed.csv\n(coords, plane, used_radius_um,\nMatch_Distance_um, areas, linear sizes)]:::artifact
    Nplot1[area_scatter_pairs.png]:::artifact
    Nplot2[linear_scatter_pairs.png]:::artifact

    Nin1 --> N1 --> Nout1
    Nin2 --> N2
    Nin3 --> N2
    Nin4 --> N2 --> N3 --> Nout2 --> N4 --> Nout3
    N4 --> Nplot1
    N4 --> Nplot2
  end

  %% Swimlane 3: pullonepair.py
  subgraph S3[pullonepair.py]
    direction TB
    P0[Select pair\n(CLI: --serial/--cell or\npick from top growth)]:::step
    P1[Pull LIS2 images + corners\n(copy archive files)]:::step
    P2[Pull LCVIS images via anapy\n(CroppedImage rectification)]:::step
    P3[LIS2: raw bbox + rectified bbox\n(using corners transform)]:::step
    P4[LCVIS: scale (itemlist um_per_cpx → LCOSInfo → area fallback),\nthen bbox + overlays (points/mask)]:::step
    P5[Optional: make growth plots\nfrom matched_pairs_detailed.csv]:::step

    Pin1[matched_pairs_detailed.csv]:::artifact
    Pin2[LIS2_{PART}_matched_defects.csv]:::artifact
    Pin3[LCVIS_{PART}_itemlists.csv]:::artifact

    Pout1[pull_summary.json / .csv]:::artifact
    Pout2[bbox_LIS2_annotated.png,\nbbox_LIS2_cutout.png]:::artifact
    Pout3[bbox_LIS2_rect_annotated.png,\nbbox_LIS2_rect_cutout.png]:::artifact
    Pout4[bbox_LCVIS_full_annotated.png,\nbbox_LCVIS_cutout.png]:::artifact
    Pout5[overlay_LIS2_raw.png,\noverlay_LIS2_rect.png,\noverlay_LCVIS_panel_points.png,\noverlay_LCVIS_panel_mask.png]:::artifact
    Pplot1[growth_area_scatter.png]:::artifact
    Pplot2[growth_linear_scatter.png]:::artifact
    Ptab1[growth_linear_table.csv]:::artifact

    Pin1 --> P0
    Pin2 --> P0
    Pin3 --> P2
    P0 --> P1 --> P3 --> Pout2 & Pout3
    P0 --> P2 --> P4 --> Pout4 & Pout5
    P0 --> P5 --> Pplot1 & Pplot2 & Ptab1
    P0 --> Pout1
  end

  %% Handoffs between lanes
  Mout1 & Mout2 & Mout3 & Mout4 -->|OUT_ROOT| Nin1 & Nin2_
