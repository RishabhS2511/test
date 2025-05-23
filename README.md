# **Code**

### **ConvexHull**

```
package view;

import org.jxmapviewer.viewer.GeoPosition;

import java.util.*;

public class ConvexHull {

    public static List<GeoPosition> getConvexHull(List<GeoPosition> points) {
        if (points.size() < 3) return new ArrayList<>();

        List<GeoPosition> sorted = new ArrayList<>(points);
        sorted.sort(Comparator.comparingDouble(GeoPosition::getLongitude)
                .thenComparingDouble(GeoPosition::getLatitude));

        List<GeoPosition> lower = new ArrayList<>();
        for (GeoPosition p : sorted) {
            while (lower.size() >= 2 && cross(lower.get(lower.size() - 2), lower.get(lower.size() - 1), p) <= 0) {
                lower.remove(lower.size() - 1);
            }
            lower.add(p);
        }

        List<GeoPosition> upper = new ArrayList<>();
        Collections.reverse(sorted);
        for (GeoPosition p : sorted) {
            while (upper.size() >= 2 && cross(upper.get(upper.size() - 2), upper.get(upper.size() - 1), p) <= 0) {
                upper.remove(upper.size() - 1);
            }
            upper.add(p);
        }

        lower.remove(lower.size() - 1);
        upper.remove(upper.size() - 1);
        lower.addAll(upper);

        return lower;
    }

    private static double cross(GeoPosition o, GeoPosition a, GeoPosition b) {
        return (a.getLongitude() - o.getLongitude()) * (b.getLatitude() - o.getLatitude()) -
                (a.getLatitude() - o.getLatitude()) * (b.getLongitude() - o.getLongitude());
    }
}

```

### Mapa Code
```
// Mapa Code

import org.jxmapviewer.JXMapViewer;
import org.jxmapviewer.OSMTileFactoryInfo;
import org.jxmapviewer.viewer.DefaultTileFactory;
import org.jxmapviewer.viewer.GeoPosition;
import org.jxmapviewer.painter.Painter;
import view.ConvexHull;

import javax.swing.*;
import java.awt.*;
import java.awt.event.MouseAdapter;
import java.awt.event.MouseEvent;
import java.awt.geom.Point2D;
import java.util.ArrayList;
import java.util.List;

public class Mapa {
    private static JXMapViewer mapViewer;
    private static List<GeoPosition> points = new ArrayList<>();

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            JFrame frame = new JFrame("Map with Convex Hull Loop");
            frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
            frame.setSize(800, 600);

            // Setup map viewer
            mapViewer = new JXMapViewer();
            OSMTileFactoryInfo info = new OSMTileFactoryInfo();
            DefaultTileFactory tileFactory = new DefaultTileFactory(info);
            mapViewer.setTileFactory(tileFactory);
            mapViewer.setZoom(4);
            mapViewer.setAddressLocation(new GeoPosition(20.5937, 78.9629)); // India center

            // Add mouse listener to add points on left click
            mapViewer.addMouseListener(new MouseAdapter() {
                @Override
                public void mouseClicked(MouseEvent e) {
                    if (SwingUtilities.isLeftMouseButton(e)) {
                        Point2D worldPos = e.getPoint();
                        GeoPosition geo = mapViewer.convertPointToGeoPosition(worldPos);
                        points.add(geo);
                        updateMapPainter();
                    }
                }
            });

            frame.add(mapViewer);
            frame.setVisible(true);
        });
    }

    private static void updateMapPainter() {
        // If less than 3 points, just draw points
        Painter<JXMapViewer> painter = (g, map, w, h) -> {
            g = (Graphics2D) g.create();

            // Draw points
            g.setColor(Color.RED);
            for (GeoPosition gp : points) {
                Point2D pt = mapViewer.convertGeoPositionToPoint(gp);
                g.fillOval((int) pt.getX() - 5, (int) pt.getY() - 5, 10, 10);
            }

            // Draw polygon if 3 or more points
            if (points.size() >= 3) {
                List<GeoPosition> hull = ConvexHull.getConvexHull(points);

                g.setColor(new Color(0, 0, 255, 100)); // translucent blue fill
                int[] xPoints = new int[hull.size()];
                int[] yPoints = new int[hull.size()];

                for (int i = 0; i < hull.size(); i++) {
                    Point2D pt = mapViewer.convertGeoPositionToPoint(hull.get(i));
                    xPoints[i] = (int) pt.getX();
                    yPoints[i] = (int) pt.getY();
                }

                g.fillPolygon(xPoints, yPoints, hull.size());

                // Draw polygon border
                g.setColor(Color.BLUE);
                g.setStroke(new BasicStroke(2));
                g.drawPolygon(xPoints, yPoints, hull.size());
            }
            g.dispose();
        };

        mapViewer.setOverlayPainter(painter);
        mapViewer.repaint();
    }
}


```

### **MarkerPainter**
```
package view;

import org.jxmapviewer.JXMapViewer;
import org.jxmapviewer.painter.Painter;
import org.jxmapviewer.viewer.GeoPosition;

import java.awt.*;
import java.awt.geom.Ellipse2D;
import java.awt.geom.Point2D;
import java.util.List;

public class MarkerPainter implements Painter<JXMapViewer> {
    private final List<GeoPosition> points;

    public MarkerPainter(List<GeoPosition> points) {
        this.points = points;
    }

    @Override
    public void paint(Graphics2D g, JXMapViewer map, int w, int h) {
        g = (Graphics2D) g.create();
        g.setColor(Color.BLUE);

        for (GeoPosition gp : points) {
            Point2D pt = map.getTileFactory().geoToPixel(gp, map.getZoom());
            int size = 10;
            int x = (int) pt.getX() - size / 2;
            int y = (int) pt.getY() - size / 2;
            g.fill(new Ellipse2D.Double(x, y, size, size));
        }

        g.dispose();
    }
}


```

### **PolygonPainter**
```

package view;

import org.jxmapviewer.JXMapViewer;
import org.jxmapviewer.painter.Painter;
import org.jxmapviewer.viewer.GeoPosition;

import java.awt.*;
import java.awt.geom.Path2D;
import java.awt.geom.Point2D;
import java.util.List;

public class PolygonPainter implements Painter<JXMapViewer> {
    private final List<GeoPosition> polygon;
    private Color color = Color.RED;
    private Color fillColor = new Color(255, 0, 0, 40);

    public PolygonPainter(List<GeoPosition> polygon) {
        this.polygon = polygon;
    }

    public void setColor(Color color) {
        this.color = color;
    }

    public void setFillColor(Color fillColor) {
        this.fillColor = fillColor;
    }

    @Override
    public void paint(Graphics2D g, JXMapViewer map, int w, int h) {
        if (polygon == null || polygon.size() < 3) return;

        Path2D path = new Path2D.Double();
        boolean first = true;
        for (GeoPosition gp : polygon) {
            Point2D pt = map.getTileFactory().geoToPixel(gp, map.getZoom());
            if (first) {
                path.moveTo(pt.getX(), pt.getY());
                first = false;
            } else {
                path.lineTo(pt.getX(), pt.getY());
            }
        }
        path.closePath();

        g.setColor(fillColor);
        g.fill(path);

        g.setColor(color);
        g.setStroke(new BasicStroke(2));
        g.draw(path);
    }
}

```

### **RoutePainter**

```
package view;

import org.jxmapviewer.JXMapViewer;
import org.jxmapviewer.painter.Painter;
import org.jxmapviewer.viewer.GeoPosition;

import java.awt.*;
import java.awt.geom.Point2D;
import java.util.List;

public class RoutePainter implements Painter<JXMapViewer> {
    private final List<GeoPosition> track;

    public RoutePainter(List<GeoPosition> track) {
        this.track = track;
    }

    @Override
    public void paint(Graphics2D g, JXMapViewer map, int w, int h) {
        g = (Graphics2D) g.create();
        g.setColor(Color.BLUE);
        g.setStroke(new BasicStroke(2));

        Point2D prev = null;
        for (GeoPosition gp : track) {
            Point2D pt = map.getTileFactory().geoToPixel(gp, map.getZoom());
            if (prev != null) {
                g.drawLine((int) prev.getX(), (int) prev.getY(), (int) pt.getX(), (int) pt.getY());
            }
            prev = pt;
        }

        g.dispose();
    }
}

```
