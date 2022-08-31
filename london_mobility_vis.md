---
id: litvis

narrative-schemas:
  - ../Litvis/mobility.yml

elm:
  dependencies:
    gicentre/elm-vegalite: latest
    gicentre/tidy: latest
---

@import "../Litvis/datavis.less"

```elm {l=hidden}
import Tidy exposing (..)
import VegaLite exposing (..)
```

### Urban mobility in London during COVID-19 pandemic

```elm {v interactive}
integratedLines : Spec
integratedLines =
    let
        cfg =
            configure
                << configuration (coView [ vicoStroke Nothing ])
                << configuration (coAxis [ axcoDomain False ])

        data =
            dataFromUrl "https://clareho.github.io/dissertation/merged_dataset_rolling.csv" []

        encLineInt =
            encoding
                << position X
                    [ pName "Date"
                    , pScale [ scDomain (doMinDt [ dtYear 2020, dtMonth Feb, dtDate 15 ]) ]
                    , pTemporal
                    , pAxis
                        [ axTitle ""
                        , axLabelAlign haLeft
                        , axLabelExpr
                            "[timeFormat(datum.value, '%b') , (timeFormat(datum.value, '%m') == '01' || (timeFormat(datum.value, '%m') == '03' && timeFormat(datum.value, '%Y') == '2020')) ? timeFormat(datum.value, '%Y') : '']"
                        , axLabelOffset 0
                        , axLabelPadding 5
                        , axTickSize 0
                        , axTickColor "#ccc"
                        , axGrid False
                        ]
                    ]

        -- Google data layer
        transLine =
            transform
                << filter (fiExpr "datum.variable == 'Retail & recreation' || datum.variable == 'Grocery & pharmacy' || datum.variable == 'Parks' || datum.variable == 'Transit stations' || datum.variable == 'Workplaces' || datum.variable == 'Residential'")

        encLine =
            encoding
                << position Y
                    [ pName "value"
                    , pQuant
                    , pScale [ scDomain (doNums [ -100, 100 ]) ]
                    , pAxis
                        [ axTitle "Precentage change from baseline (%)"
                        , axValues (nums [ -100, 0, 100 ])
                        , axTicks False
                        ]
                    ]
                << color
                    [ mCondition (prParam "mySelection")
                        [ mName "variable"
                        , mTitle "[Please click below] \nCategories:"
                        , mScale [ scScheme "tableau10" [] ]
                        ]
                        [ mStr "grey" ]
                    ]
                << opacity
                    [ mCondition (prParam "mySelection")
                        [ mNum 1 ]
                        [ mNum 0.1 ]
                    ]
                << tooltips
                    [ [ tName "Date", tTimeUnit yearMonthDate, tFormat "%Y-%b-%d", tTitle "Date" ]
                    , [ tName "variable", tTitle "Categories" ]
                    , [ tName "value", tFormat ".0f", tTitle "% change" ]
                    ]

        ps =
            params
                << param "mySelection"
                    [ paSelect sePoint [ seFields [ "variable" ] ]
                    , paBindLegend "click"
                    ]

        specLine =
            asSpec [ line [], transLine [], encLine [], ps [] ]

        -- Stringency index layer
        transStr =
            transform
                << filter (fiExpr "datum.variable == 'Stringency Index'")

        encStr =
            encoding
                << position Y
                    [ pName "value"
                    , pQuant
                    , pScale [ scDomain (doNums [ -100, 100 ]) ]
                    , pAxis
                        [ axTitle "Stringency Index"
                        , axLabelColor "#7B7D7D"
                        , axTitleColor "#7B7D7D"
                        , axValues (nums [ -100, 0, 100 ])
                        , axDataCondition (expr "datum.value == 0")
                            (cAxGridColor "#000" "#ccc")
                        , axDataCondition (expr "datum.value == 0")
                            (cAxGridWidth 2 0.5)
                        , axGrid True
                        , axTicks False
                        ]
                    ]

        specStr =
            asSpec [ area [ maColor "#999", maOpacity 0.3 ], transStr [], encStr [] ]

        res =
            resolve
                << resolution (reScale [ ( chY, reIndependent ) ])

        specGlStr =
            asSpec [ layer [ specLine, specStr ], res [] ]

        -- Annotation layer 1
        annotationData =
            dataFromUrl "https://clareho.github.io/dissertation/restrictions_summary_20220706r.csv" []

        encAnnotation1 =
            encoding
                << position X
                    [ pName "date"
                    , pTemporal
                    ]
                << position Y
                    [ pName "position"
                    , pScale [ scDomain (doNums [ -100, 100 ]) ]
                    , pQuant
                    , pAxis []
                    ]
                << text [ tName "restriction" ]

        specAnnotation1 =
            asSpec [ annotationData, encAnnotation1 [], textMark [ maDy 0, maAngle -70, maAlign haLeft, maFontSize 9 ] ]

        -- Annotation layer 2
        encAnnotation2 =
            encoding
                << position X
                    [ pName "date"
                    , pTemporal
                    ]
                << position Y
                    [ pName "position"
                    , pScale [ scDomain (doNums [ -100, 100 ]) ]
                    , pQuant
                    , pAxis []
                    ]

        specAnnotation2 =
            asSpec [ annotationData, encAnnotation2 [], rule [ maColor "grey", maOpacity 0.5, maStrokeDash [ 2, 2 ] ] ]

        -- Annotation layer 3
        encAnnotation3 =
            encoding
                << position X
                    [ pName "date"
                    , pTemporal
                    ]
                << position Y
                    [ pName "bottom"
                    , pScale [ scDomain (doNums [ -100, 100 ]) ]
                    , pQuant
                    , pAxis []
                    ]

        specAnnotation3 =
            asSpec [ annotationData, encAnnotation3 [], rule [ maColor "grey", maOpacity 0.5, maStrokeDash [ 2, 2 ] ] ]

        specAnnotation =
            asSpec [ layer [ specAnnotation1, specAnnotation2, specAnnotation3 ] ]

        specLineInt =
            asSpec [ width 800, height 400, encLineInt [], layer [ specAnnotation, specGlStr ] ]

        -- Area chart Covid cases
        casesData =
            dataFromUrl "https://clareho.github.io/dissertation/region_2022-07-06_london_cases_format.csv" []

        encCases =
            encoding
                << position X
                    [ pName "date"
                    , pScale [ scDomain (doMinDt [ dtYear 2020, dtMonth Feb, dtDate 15 ]) ]
                    , pTemporal
                    , pAxis
                        [ axTitle ""
                        , axLabelAlign haLeft
                        , axLabelExpr
                            "[timeFormat(datum.value, '%b') , (timeFormat(datum.value, '%m') == '01' || (timeFormat(datum.value, '%m') == '03' && timeFormat(datum.value, '%Y') == '2020')) ? timeFormat(datum.value, '%Y') : '']"
                        , axLabelOffset 0
                        , axLabelPadding 5
                        , axTickSize 0
                        , axTickColor "#ccc"
                        , axGrid False
                        ]
                    ]
                << tooltips
                    [ [ tName "date", tTimeUnit yearMonthDate, tFormat "%Y-%b-%d", tTitle "Date" ]
                    , [ tName "New Cases", tFormat ",d", tTitle "No. of new cases" ]
                    , [ tName "Death Cases", tFormat ",d", tTitle "No. of death cases" ]
                    ]

        encNewCases =
            encoding
                << position Y
                    [ pName "New Cases"
                    , pQuant
                    , pAxis
                        [ axTitle "No. of new cases"
                        , axTicks False
                        , axGrid False
                        , axTitleColor "#1927B0"
                        , axLabelColor "#1927B0"
                        ]
                    ]

        specNewCases =
            asSpec [ casesData, encNewCases [], area [ maColor "blue", maOpacity 0.5 ] ]

        encDeathCases =
            encoding
                << position Y
                    [ pName "Death Cases"
                    , pQuant
                    , pAxis
                        [ axTitle "No. of death cases"
                        , axTicks False
                        , axGrid False
                        , axTitleColor "#DC4F4F"
                        , axLabelColor "#DC4F4F"
                        ]
                    ]

        specDeathCases =
            asSpec [ casesData, encDeathCases [], area [ maColor "red", maOpacity 0.3 ] ]

        specCases =
            asSpec [ encCases [], layer [ specNewCases, specDeathCases ], res [] ]

        -- Annotation layer 5
        encAnnotation5 =
            encoding
                << position X
                    [ pName "date"
                    , pTemporal
                    ]
                << position Y
                    [ pName "top"
                    , pScale [ scDomain (doNums [ 0, 100 ]) ]
                    , pQuant
                    , pAxis []
                    ]

        specAnnotation5 =
            asSpec [ annotationData, encAnnotation5 [], rule [ maColor "grey", maOpacity 0.8, maStrokeDash [ 2, 2 ] ] ]

        specCasesInt =
            asSpec [ width 800, height 100, layer [ specCases, specAnnotation5 ] ]

        -- Heatmap layer
        transHeat =
            transform
                << filter (fiExpr "datum.variable == 'Retail & recreation' || datum.variable == 'Grocery & pharmacy' || datum.variable == 'Parks' || datum.variable == 'Transit stations' || datum.variable == 'Workplaces' || datum.variable == 'Residential'")

        encHeat =
            encoding
                << position X
                    [ pName "Date"
                    , pOrdinal
                    , pTimeUnit yearMonthDate
                    , pScale [ scDomain (doMinDt [ dtYear 2020, dtMonth Feb, dtDate 15 ]) ]
                    , pAxis
                        [ axTitle ""
                        , axFormat "%b"
                        , axLabelAngle 0
                        , axLabelOverlap osNone
                        , axLabelAlign haLeft
                        , axDataCondition
                            (fiEqual "value" (dt [ dtDate 1 ]) |> fiOpTrans (mTimeUnit date))
                            (cAxLabelColor "black" "")
                        , axDataCondition
                            (fiEqual "value" (dt [ dtDate 1 ]) |> fiOpTrans (mTimeUnit date))
                            (cAxTickColor "black" "")
                        , axLabelExpr
                            "[timeFormat(datum.value, '%b') , (timeFormat(datum.value, '%m') == '01' || (timeFormat(datum.value, '%m') == '03' && timeFormat(datum.value, '%Y') == '2020')) ? timeFormat(datum.value, '%Y') : '']"
                        , axTickSize 0
                        , axLabelOffset 0
                        , axLabelPadding 5
                        , axTickColor "#ccc"
                        ]
                    ]
                << position Y
                    [ pName "variable"
                    , pTitle ""
                    , pAxis
                        [ axTicks False
                        , axLabelPadding 5
                        ]
                    ]
                << color
                    [ mAggregate opSum
                    , mName "value"
                    , mTitle "% change"
                    , mLegend [ leGradientLength 80, leGradientThickness 10 ]
                    , mScale
                        [ scScheme "redblue" []
                        , scDomain (doMid 0)
                        ]
                    ]
                << tooltips
                    [ [ tName "Date", tTimeUnit yearMonthDate, tFormat "%Y-%b-%d", tTitle "Date" ]
                    , [ tName "variable", tTitle "Categories" ]
                    , [ tName "value", tFormat ".0f", tTitle "% change" ]
                    ]

        specHeat =
            asSpec [ rect [], transHeat [], encHeat [] ]

        -- Annotation layer 4
        encAnnotation4 =
            encoding
                << position X
                    [ pName "date"
                    , pTemporal
                    ]
                << position Y
                    [ pName "top"
                    , pScale [ scDomain (doNums [ 0, 100 ]) ]
                    , pQuant
                    , pAxis []
                    ]

        specAnnotation4 =
            asSpec [ annotationData, encAnnotation4 [], rule [ maColor "grey", maOpacity 0.8, maStrokeDash [ 2, 2 ] ] ]

        specHeatInt =
            asSpec [ width 800, height 100, layer [ specHeat ] ]
    in
    toVegaLite [ data, cfg [], spacing 15, vConcat [ specLineInt, specCasesInt, specHeatInt ] ]
```

### Urban Mobility in London Boroughs

```elm {v interactive}
choropleth_glb : Spec
choropleth_glb =
    let
        cfg =
            configure
                << configuration (coView [ vicoStroke Nothing ])
                << configuration (coAxis [ axcoDomain False ])

        glbData =
            dataFromUrl "https://clareho.github.io/dissertation/google_mobility_london_boroughs_2.csv" []

        ladData =
            dataFromUrl "https://clareho.github.io/dissertation/london_borough_boundaries.json" [ topojsonFeature "LAD_DEC_2021_GB_BFC" ]

        transGlb =
            transform
                << filter (fiExpr "year(datum.date) == yearSlider")
                << filter (fiExpr "dayofyear(datum.date) == doySlider")
                << lookup "borough" ladData "properties.LAD21NM" (luAs "geo")

        psGlb =
            params
                << param "yearSlider"
                    [ paValue (num 2020)
                    , paBind (ipRange [ inName "Year", inMin 2020, inMax 2022, inStep 1 ])
                    ]
                << param "doySlider"
                    [ paValue (num 50)
                    , paBind (ipRange [ inName "Day ", inMin 1, inMax 366, inStep 1 ])
                    ]

        specPoly =
            asSpec
                [ ladData
                , geoshape [ maFill "", maStroke "white", maStrokeWidth 0.1, maOpacity 1 ]
                ]

        -- Grocery & pharmacy facet
        encGlbGP =
            encoding
                << color
                    [ mName "Grocery & pharmacy"
                    , mQuant
                    , mScale
                        [ scScheme "redblue" []
                        , scDomain (doMid 0)
                        , scDomain (doMin -100)
                        , scDomain (doMax 50)
                        ]
                    , mLegend
                        [ leGradientLength 80
                        , leGradientThickness 8
                        , leLabelFontSize 8
                        , leTitle "% change"
                        , leTitleFontSize 8
                        , leOrient loRight
                        ]
                    ]
                << shape [ mName "geo", mGeo ]
                << tooltips
                    [ [ tName "Grocery & pharmacy", tFormat ".0f", tTitle "% change" ]
                    , [ tName "borough", tTitle "Borough" ]
                    ]

        specGlbGP =
            asSpec
                [ width 250
                , height 200
                , glbData
                , transGlb []
                , encGlbGP []
                , geoshape [ maStroke "white", maStrokeWidth 0.2 ]
                , title "Grocery & pharmacy" [ tiFontSize 10 ]
                ]

        -- Retail & recreation facet
        encGlbRP =
            encoding
                << color
                    [ mName "Retail & recreation"
                    , mQuant
                    , mScale
                        [ scScheme "redblue" []
                        , scDomain (doMid 0)
                        , scDomain (doMin -100)
                        , scDomain (doMax 50)
                        ]
                    , mLegend
                        [ leGradientLength 80
                        , leGradientThickness 8
                        , leLabelFontSize 8
                        , leTitle "% change"
                        , leTitleFontSize 8
                        , leOrient loRight
                        ]
                    ]
                << shape [ mName "geo", mGeo ]
                << tooltips
                    [ [ tName "Retail & recreation", tFormat ".0f", tTitle "% change" ]
                    , [ tName "borough", tTitle "Borough" ]
                    ]

        specGlbRP =
            asSpec
                [ width 250
                , height 200
                , glbData
                , transGlb []
                , encGlbRP []
                , geoshape [ maStroke "white", maStrokeWidth 0.2 ]
                , title "Retail & recreation" [ tiFontSize 10 ]
                ]

        -- Residential facet
        encGlbR =
            encoding
                << color
                    [ mName "Residential"
                    , mQuant
                    , mScale
                        [ scScheme "redblue" []
                        , scDomain (doMid 0)
                        , scDomain (doMin -100)
                        , scDomain (doMax 50)
                        ]
                    , mLegend
                        [ leGradientLength 80
                        , leGradientThickness 8
                        , leLabelFontSize 8
                        , leTitle "% change"
                        , leTitleFontSize 8
                        , leOrient loRight
                        ]
                    ]
                << shape [ mName "geo", mGeo ]
                << tooltips
                    [ [ tName "Residential", tFormat ".0f", tTitle "% change" ]
                    , [ tName "borough", tTitle "Borough" ]
                    ]

        specGlbR =
            asSpec
                [ width 250
                , height 200
                , glbData
                , transGlb []
                , encGlbR []
                , geoshape [ maStroke "white", maStrokeWidth 0.2 ]
                , title "Residential" [ tiFontSize 10 ]
                ]

        -- Workplaces facet
        encGlbW =
            encoding
                << color
                    [ mName "Workplaces"
                    , mQuant
                    , mScale
                        [ scScheme "redblue" []
                        , scDomain (doMid 0)
                        , scDomain (doMin -100)
                        , scDomain (doMax 50)
                        ]
                    , mLegend
                        [ leGradientLength 80
                        , leGradientThickness 8
                        , leLabelFontSize 8
                        , leTitle "% change"
                        , leTitleFontSize 8
                        , leOrient loRight
                        ]
                    ]
                << shape [ mName "geo", mGeo ]
                << tooltips
                    [ [ tName "Workplaces", tFormat ".0f", tTitle "% change" ]
                    , [ tName "borough", tTitle "Borough" ]
                    ]

        specGlbW =
            asSpec
                [ width 250
                , height 200
                , glbData
                , transGlb []
                , encGlbW []
                , geoshape [ maStroke "white", maStrokeWidth 0.2 ]
                , title "Workplaces" [ tiFontSize 10 ]
                ]

        -- Transport facet
        encGlbTS =
            encoding
                << color
                    [ mName "Transit stations"
                    , mQuant
                    , mScale
                        [ scScheme "redblue" []
                        , scDomain (doMid 0)
                        , scDomain (doMin -100)
                        , scDomain (doMax 50)
                        ]
                    , mLegend
                        [ leGradientLength 80
                        , leGradientThickness 8
                        , leLabelFontSize 8
                        , leTitle "% change"
                        , leTitleFontSize 8
                        , leOrient loRight
                        ]
                    ]
                << shape [ mName "geo", mGeo ]
                << tooltips
                    [ [ tName "Transit stations", tFormat ".0f", tTitle "% change" ]
                    , [ tName "borough", tTitle "Borough" ]
                    ]

        specGlbTS =
            asSpec
                [ width 250
                , height 200
                , glbData
                , transGlb []
                , encGlbTS []
                , geoshape [ maStroke "white", maStrokeWidth 0.2 ]
                , title "Transit station" [ tiFontSize 10 ]
                ]

        -- Parks facet
        encGlbP =
            encoding
                << color
                    [ mName "Parks"
                    , mQuant
                    , mScale
                        [ scScheme "redblue" []
                        , scDomain (doMid 0)
                        , scDomain (doMin -100)
                        , scDomain (doMax 300)
                        ]
                    , mLegend
                        [ leGradientLength 80
                        , leGradientThickness 8
                        , leLabelFontSize 8
                        , leTitle "% change"
                        , leTitleFontSize 8
                        , leOrient loRight
                        ]
                    ]
                << shape [ mName "geo", mGeo ]
                << tooltips
                    [ [ tName "Parks", tFormat ".0f", tTitle "% change" ]
                    , [ tName "borough", tTitle "Borough" ]
                    ]

        specGlbP =
            asSpec
                [ width 250
                , height 200
                , glbData
                , transGlb []
                , encGlbP []
                , geoshape [ maStroke "white", maStrokeWidth 0.2 ]
                , title "Parks" [ tiFontSize 10 ]
                ]

        -- All facets
        res =
            resolve
                << resolution (reScale [ ( chColor, reIndependent ) ])

        specGlbV1 =
            asSpec [ res [], vConcat [ specGlbGP, specGlbP, specGlbR ] ]

        specGlbV2 =
            asSpec [ res [], vConcat [ specGlbRP, specGlbTS, specGlbW ] ]

        specGlbPolyAll =
            asSpec [ res [], hConcat [ specGlbV1, specGlbV2 ] ]

        -- Stringency level donut chart
        dataStrPie =
            dataFromUrl "https://clareho.github.io/dissertation/Stringency_index.csv" []

        transStrPie =
            transform
                << filter (fiExpr "year(datum.date) == yearSlider")
                << filter (fiExpr "dayofyear(datum.date) == doySlider")

        colorStr =
            categoricalDomainMap
                [ ( "Stringency Index", "rgb(0,139,139)" )
                , ( "inverted_index", "rgb(248,248,255)" )
                ]

        encStrPie =
            encoding
                << position Theta
                    [ pName "value"
                    , pQuant
                    ]
                << color
                    [ mName "variable", mScale colorStr, mLegend [] ]

        specStrPie =
            asSpec [ arc [ maInnerRadius 55, maOuterRadius 75 ] ]

        transPieLabel =
            transform
                << filter (fiExpr "datum.variable == 'Stringency Index' ")

        encPieLabel =
            encoding
                << position X
                    [ pNum 100 ]
                << position Y
                    [ pNum 100 ]
                << text [ tName "value", tFormat ".0f" ]

        specPieLabel =
            asSpec [ encPieLabel [], transPieLabel [], textMark [ maFontSize 18, maFontStyle "bold" ] ]

        specStrPieInt =
            asSpec
                [ dataStrPie
                , transStrPie []
                , encStrPie []
                , layer [ specStrPie, specPieLabel ]
                , title "Stringency Index" [ tiFontSize 10 ]
                ]

        -- Title layer
        annotationData =
            dataFromUrl "https://clareho.github.io/dissertation/restrictions_daily_20220706_format.csv" []

        encDate =
            encoding
                << position X
                    [ pNum 100 ]
                << position Y
                    [ pNum 180 ]
                << text [ tName "date", tTemporal ]

        specDate =
            asSpec
                [ width 250
                , height 200
                , encDate []
                , glbData
                , transGlb []
                , textMark [ maFontSize 20 ]
                ]

        encRestrict =
            encoding
                << position X
                    [ pNum 100 ]
                << position Y
                    [ pNum 10 ]
                << text [ tName "restriction" ]

        transRestrict =
            transform
                << filter (fiExpr "year(datum.date) == yearSlider")
                << filter (fiExpr "dayofyear(datum.date) == doySlider")

        specRestrict =
            asSpec
                [ width 250
                , height 200
                , encRestrict []
                , annotationData
                , transRestrict []
                , textMark [ maFontSize 12 ]
                ]

        specTitle =
            asSpec
                [ vConcat [ specDate, specRestrict ] ]

        specTitleStr =
            asSpec [ vConcat [ specStrPieInt, specTitle ] ]
    in
    toVegaLite
        [ cfg []
        , psGlb []
        , res []
        , hConcat [ specTitleStr, specGlbPolyAll ]
        ]
```
