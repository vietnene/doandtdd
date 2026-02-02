import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.util.ArrayList;
import java.util.prefs.Preferences;

public class FlappyBirdGame extends JPanel implements ActionListener, KeyListener {
    // --- HỆ THỐNG LƯU TRỮ (CSDL - CRUD) ---
    private int score = 0;
    private int highScore = 0;
    private final Preferences prefs = Preferences.userNodeForPackage(FlappyBirdGame.class);
    
    // --- BIẾN TRÒ CHƠI ---
    private double birdY; 
    private double velocityY = 0;
    private final double gravity = 0.5;
    private final ArrayList<Rectangle2D_Custom> pipes = new ArrayList<>();
    private boolean isGameOver = false;
    private final Timer gameLoop;
    private final Timer pipeTimer;

    public FlappyBirdGame() {
        highScore = prefs.getInt("highScore", 0);
        setBackground(new Color(135, 206, 235));
        setFocusable(true);
        addKeyListener(this);

        // Vòng lặp 60 FPS
        gameLoop = new Timer(1000 / 60, this);
        gameLoop.start();

        // Cứ 1.5 giây tạo ống mới
        pipeTimer = new Timer(1500, e -> createPipe());
        pipeTimer.start();
    }

    // Lớp hỗ trợ lưu tỷ lệ vị trí ống nước thay vì tọa độ cứng
    class Rectangle2D_Custom {
        double xRel, yRel, wRel, hRel;
        boolean passed = false;

        Rectangle2D_Custom(double x, double y, double w, double h) {
            this.xRel = x; this.yRel = y; this.wRel = w; this.hRel = h;
        }
    }

    private void createPipe() {
        if (getHeight() <= 0) return;
        double gapRel = 0.25; // Khoảng trống bằng 25% chiều cao
        double pipeWRel = 0.15; // Độ rộng ống bằng 15% chiều rộng
        double randomYRel = -0.5 + Math.random() * 0.4; // Ngẫu nhiên vị trí ống trên

        // Lưu theo tỷ lệ (từ 0.0 đến 1.0)
        pipes.add(new Rectangle2D_Custom(1.0, randomYRel, pipeWRel, 0.7)); // Ống trên
        pipes.add(new Rectangle2D_Custom(1.0, randomYRel + 0.7 + gapRel, pipeWRel, 0.7)); // Ống dưới
    }

    private void move() {
        if (getHeight() <= 0) return;

        velocityY += gravity;
        birdY += velocityY / getHeight(); // Tốc độ rơi theo tỷ lệ

        for (int i = 0; i < pipes.size(); i++) {
            Rectangle2D_Custom p = pipes.get(i);
            p.xRel -= 0.01; // Di chuyển 1% chiều rộng mỗi khung hình

            // Tính va chạm (Sử dụng tọa độ thực tế để check)
            double bSize = getHeight() * 0.06;
            double bX = getWidth() / 8.0;
            double bY = birdY * getHeight();
            
            double pX = p.xRel * getWidth();
            double pY = p.yRel * getHeight();
            double pW = p.wRel * getWidth();
            double pH = p.hRel * getHeight();

            if (new Rectangle((int)bX, (int)bY, (int)bSize, (int)bSize).intersects(new Rectangle((int)pX, (int)pY, (int)pW, (int)pH))) {
                endGame();
            }

            // Tính điểm
            if (!p.passed && pX + pW < bX) {
                p.passed = true;
                if (i % 2 == 0) {
                    score++;
                    if (score > highScore) {
                        highScore = score;
                        prefs.putInt("highScore", highScore);
                    }
                }
            }
        }

        if (birdY > 1.0 || birdY < -0.1) endGame();
        pipes.removeIf(p -> p.xRel < -0.2);
    }

    private void endGame() {
        isGameOver = true;
        gameLoop.stop();
        pipeTimer.stop();
    }

    @Override
    protected void paintComponent(Graphics g) {
        super.paintComponent(g);
        int w = getWidth();
        int h = getHeight();
        if (h <= 0) return;

        // Vẽ Chim (Tỷ lệ 6% chiều cao)
        int bSize = (int)(h * 0.06);
        int bX = w / 8;
        int bY = (int)(birdY * h);
        g.setColor(Color.YELLOW);
        g.fillOval(bX, bY, bSize, bSize);

        // Vẽ Cột (Tỷ lệ theo màn hình)
        g.setColor(new Color(34, 139, 34));
        for (Rectangle2D_Custom p : pipes) {
            g.fillRect((int)(p.xRel * w), (int)(p.yRel * h), (int)(p.wRel * w), (int)(p.hRel * h));
        }

        // UI
        g.setColor(Color.WHITE);
        g.setFont(new Font("Arial", Font.BOLD, h / 25));
        g.drawString("Điểm: " + score + " | Kỷ lục: " + highScore, 20, h / 15);
        
        if (isGameOver) {
            g.setColor(new Color(0, 0, 0, 150));
            g.fillRect(0, 0, w, h);
            g.setColor(Color.WHITE);
            g.drawString("GAME OVER", w/2 - 50, h/2);
            g.setFont(new Font("Arial", Font.PLAIN, h/40));
            g.drawString("SPACE: Chơi lại | R: Xóa kỷ lục", w/2 - 80, h/2 + 50);
        }
    }

    @Override
    public void actionPerformed(ActionEvent e) {
        if (!isGameOver) move();
        repaint();
    }

    @Override
    public void keyPressed(KeyEvent e) {
        if (e.getKeyCode() == KeyEvent.VK_SPACE) {
            if (isGameOver) {
                score = 0; birdY = 0.5; velocityY = 0; pipes.clear();
                isGameOver = false; gameLoop.start(); pipeTimer.start();
            } else {
                velocityY = -getHeight() * 0.015; // Lực nhảy tỷ lệ với độ cao
            }
        }
        if (e.getKeyCode() == KeyEvent.VK_R) {
            prefs.putInt("highScore", 0);
            highScore = 0;
            repaint();
            JOptionPane.showMessageDialog(this, "Đã xóa kỷ lục!");
        }
    }

    @Override public void keyTyped(KeyEvent e) {}
    @Override public void keyReleased(KeyEvent e) {}

    public static void main(String[] args) {
        JFrame frame = new JFrame("Flappy Bird Adaptive UI");
        FlappyBirdGame game = new FlappyBirdGame();
        frame.add(game);
        frame.setExtendedState(JFrame.MAXIMIZED_BOTH); // Mặc định mở to
        frame.setMinimumSize(new Dimension(400, 400));
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setVisible(true);
    }
}
