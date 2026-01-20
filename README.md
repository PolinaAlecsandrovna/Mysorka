package com.example.gonkiai;

import android.os.Bundle; import android.os.Handler; import android.view.View; import android.view.ViewGroup; import android.widget.Button; import android.widget.TextView; import android.graphics.Color; import androidx.appcompat.app.AppCompatActivity;

public class MainActivity extends AppCompatActivity {

    private Button btnStart;
    private TextView tvResult;
    private View carPlayer1;
    private View carPlayer2;

    private boolean gameStarted = false;
    private boolean gameFinished = false;

    private final Handler aiHandler = new Handler();
    private final Runnable aiMoveRunnable = new Runnable() {
        @Override
        public void run() {
            if (gameStarted && !gameFinished) {
                moveAI();
                aiHandler.postDelayed(this, 200);
            }
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        btnStart = findViewById(R.id.btnStart);
        tvResult = findViewById(R.id.tvResult);
        carPlayer1 = findViewById(R.id.car_player1);
        carPlayer2 = findViewById(R.id.car_player2);
    }

    public void startGame(View view) {
        if (gameFinished) {
            resetGame();
        } else if (!gameStarted) {
            gameStarted = true;
            btnStart.setText("Пауза");
            btnStart.setBackgroundColor(Color.YELLOW);
            aiHandler.post(aiMoveRunnable);
        } else {
            gameStarted = false;
            btnStart.setText("Продолжить");
            btnStart.setBackgroundColor(Color.GREEN);
            aiHandler.removeCallbacks(aiMoveRunnable);
        }
    }

    public void drivePlayer(View view) {
        if (!gameStarted || gameFinished) return;

        ViewGroup.MarginLayoutParams player1Params = (ViewGroup.MarginLayoutParams) carPlayer1.getLayoutParams();
        player1Params.leftMargin += 50;
        carPlayer1.setLayoutParams(player1Params);

        checkWin();
    }

    private void moveAI() {
        ViewGroup.MarginLayoutParams aiParams = (ViewGroup.MarginLayoutParams) carPlayer2.getLayoutParams();
        aiParams.leftMargin += 50;
        carPlayer2.setLayoutParams(aiParams);
        checkWin();
    }

    private void checkWin() {
        int player1Position = ((ViewGroup.MarginLayoutParams) carPlayer1.getLayoutParams()).leftMargin;
        int player2Position = ((ViewGroup.MarginLayoutParams) carPlayer2.getLayoutParams()).leftMargin;

        if (player1Position >= 1700) {
            endGame("Вы победили!", Color.GREEN);
        } else if (player2Position >= 1700) {
            endGame("AI победил!", Color.RED);
        }
    }

    private void endGame(String resultText, int buttonColor) {
        gameStarted = false;
        gameFinished = true;
        aiHandler.removeCallbacks(aiMoveRunnable);
        tvResult.setText(resultText);
        btnStart.setText("Заново");
        btnStart.setBackgroundColor(buttonColor);
    }

    private void resetGame() {
        gameStarted = false;
        gameFinished = false;
        btnStart.setText("Старт");
        btnStart.setBackgroundColor(Color.RED);
        tvResult.setText("");

        resetCarPosition(carPlayer1);
        resetCarPosition(carPlayer2);
    }

    private void resetCarPosition(View car) {
        ViewGroup.MarginLayoutParams params = (ViewGroup.MarginLayoutParams) car.getLayoutParams();
        params.leftMargin = 0;
        car.setLayoutParams(params);
    }
}
        params.leftMargin = 0;
        car.setLayoutParams(params);
    }
}
