# Mysorka
package com.example.gonkiai;

import android.os.Bundle;
import android.os.Handler;
import android.view.View;
import android.view.ViewGroup;
import android.widget.Button;
import android.widget.TextView;
import android.graphics.Color;

import androidx.appcompat.app.AppCompatActivity;

public class MainActivity extends AppCompatActivity {

    private Button btnStart;
    private Button btnDrive;
    private TextView tvResult;
    private View car_player1;
    private View car_player2;

    private boolean gameStarted = false;
    private boolean gameFinished = false;

    // AI Movement Handler
    private Handler aiHandler = new Handler();
    private Runnable aiMoveRunnable = new Runnable() {
        @Override
        public void run() {
            if (gameStarted && !gameFinished) {
                moveAI();
                aiHandler.postDelayed(this, 300);
            }
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        btnStart = findViewById(R.id.btnStart);
        btnDrive = findViewById(R.id.btnDrive);
        tvResult = findViewById(R.id.tvResult);
        car_player1 = findViewById(R.id.car_player1);
        car_player2 = findViewById(R.id.car_player2);
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

        ViewGroup.MarginLayoutParams player1Params = (ViewGroup.MarginLayoutParams) car_player1.getLayoutParams();
        player1Params.leftMargin += 50;
        car_player1.setLayoutParams(player1Params);

        checkWin();
    }

    private void moveAI() {
        ViewGroup.MarginLayoutParams aiParams = (ViewGroup.MarginLayoutParams) car_player2.getLayoutParams();
        aiParams.leftMargin += 10;
        car_player2.setLayoutParams(aiParams);
        checkWin();
    }

    private void checkWin() {
        int player1Position = ((ViewGroup.MarginLayoutParams) car_player1.getLayoutParams()).leftMargin;
        int player2Position = ((ViewGroup.MarginLayoutParams) car_player2.getLayoutParams()).leftMargin;

        if (player1Position >= 800) {
            gameFinished = true;
            gameStarted = false;
            aiHandler.removeCallbacks(aiMoveRunnable);
            tvResult.setText("Вы победили!");
            btnStart.setText("Заново");
            btnStart.setBackgroundColor(Color.GREEN);
        } else if (player2Position >= 800) {
            gameFinished = true;
            gameStarted = false;
            aiHandler.removeCallbacks(aiMoveRunnable);
            tvResult.setText("AI победил!");
            btnStart.setText("Заново");
            btnStart.setBackgroundColor(Color.RED);
        }
    }

    private void resetGame() {
        gameStarted = false;
        gameFinished = false;
        btnStart.setText("Старт");
        btnStart.setBackgroundColor(Color.RED);
        tvResult.setText("");

        ViewGroup.MarginLayoutParams player1Params = (ViewGroup.MarginLayoutParams) car_player1.getLayoutParams();
        player1Params.leftMargin = 0;
        car_player1.setLayoutParams(player1Params);

        ViewGroup.MarginLayoutParams player2Params = (ViewGroup.MarginLayoutParams) car_player2.getLayoutParams();
        player2Params.leftMargin = 0;
        car_player2.setLayoutParams(player2Params);
    }
}
Field can be converted to a local variable
Private field 'btnDrive' is assigned but never accessed
Field 'aiHandler' may be 'final'
Field 'aiMoveRunnable' may be 'final'
