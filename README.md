private void btnCalculate_Click(object sender, RoutedEventArgs e)
{
    decimal price;
    decimal discountPercentage;

    if (decimal.TryParse(tbPrice.Text, out price) &&
        decimal.TryParse(tbPercent.Text, out discountPercentage))
    {
        decimal result = DiscountLogic.CalculateDiscountedPrice(price, discountPercentage);

        tbResult.Text = result.ToString();
    }
    else
    {
        MessageBox.Show("Пожалуйста, введите корректные числа");
    }
}
